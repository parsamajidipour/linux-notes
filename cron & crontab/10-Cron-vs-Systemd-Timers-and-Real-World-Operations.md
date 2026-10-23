# Cron vs Systemd Timers and Real-World Operations

Cron remains one of the simplest and most widely understood schedulers on Unix-like systems.

A line such as:

```cron
0 2 * * * /usr/local/sbin/nightly-backup
```

is easy to read.

It says:

```text
run nightly-backup every day at 02:00
```

That simplicity is valuable.

But modern Linux systems often use systemd for:

```text
service management
dependency ordering
resource controls
logging
sandboxing
startup
shutdown
timers
```

Systemd timers therefore solve some scheduling problems that cron handles only indirectly.

This does not mean cron is obsolete.

It means the two tools have different strengths.

A good Linux administrator should be able to look at a scheduled workload and decide:

```text
cron is enough
```

or:

```text
this job belongs in a systemd timer
```

based on operational requirements rather than preference.

The most useful comparison is not:

```text
which syntax looks nicer?
```

It is:

```text
which scheduler gives this workload the execution semantics it actually needs?
```

---

## The simplest possible comparison

Cron:

```cron
*/5 * * * * /opt/jobs/health-check
```

Systemd timer:

```ini
[Unit]
Description=Run health check every five minutes

[Timer]
OnCalendar=*:0/5

[Install]
WantedBy=timers.target
```

with a service:

```ini
[Unit]
Description=Application health check

[Service]
Type=oneshot
ExecStart=/opt/jobs/health-check
```

Cron expresses:

```text
schedule + command
```

in one place.

Systemd separates:

```text
timer
```

from:

```text
service execution
```

That separation initially feels more verbose.

It also gives systemd a place to define:

```text
execution user
working directory
environment
dependencies
resource limits
sandboxing
timeout
logging
restart policy
filesystem restrictions
network restrictions
```

Cron delegates most of those concerns to shell scripts and external operating-system configuration.

That is the central architectural difference.

---

## Cron is scheduler-oriented

A traditional cron entry answers:

```text
When should this shell command be launched?
```

Example:

```cron
30 1 * * * /usr/local/libexec/build-report
```

The cron daemon decides whether current calendar fields match.

Then it launches the command under the configured user context.

The command itself must handle most other concerns:

```text
current directory
environment
timeout
lock
logging
dependencies
resource limits
security hardening
retries
```

Cron is intentionally small.

That is why it has survived for decades.

---

## Systemd timers are activation-oriented

A systemd timer does not normally execute the workload directly.

It activates a unit, usually a service.

Conceptually:

```text
timer becomes due
    ->
systemd activates service
    ->
service manager applies unit configuration
    ->
process starts
```

The service can define:

```ini
User=report
Group=report
WorkingDirectory=/srv/reporting
EnvironmentFile=/etc/reporting/report.env
ExecStart=/srv/reporting/.venv/bin/python /srv/reporting/daily.py
TimeoutStartSec=20min
Nice=10
```

The timer remains responsible for:

```text
when
```

The service defines:

```text
how
```

That separation makes complex system workloads easier to reason about.

---

## A cron job with many hidden assumptions

Consider:

```cron
0 4 * * * cd /srv/app && APP_ENV=production /usr/bin/php artisan report:daily >> /var/log/app/report.log 2>&1
```

This one line contains several responsibilities:

```text
schedule
working directory
environment
interpreter
application command
output destination
error redirection
```

If the job later needs:

```text
timeout
CPU priority
memory limit
dependency on PostgreSQL
filesystem protection
network isolation
```

the crontab line becomes increasingly awkward.

A systemd service can express these separately.

---

## Equivalent systemd service

```ini
[Unit]
Description=Generate application daily report
After=postgresql.service
Wants=postgresql.service

[Service]
Type=oneshot
User=app
Group=app
WorkingDirectory=/srv/app
Environment=APP_ENV=production
ExecStart=/usr/bin/php artisan report:daily
StandardOutput=journal
StandardError=journal
TimeoutStartSec=30min
```

Timer:

```ini
[Unit]
Description=Run daily application report

[Timer]
OnCalendar=*-*-* 04:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Now the execution contract is visible without decoding shell operators.

---

## Service and timer naming

A common layout:

```text
/etc/systemd/system/company-report.service
/etc/systemd/system/company-report.timer
```

By default, a timer named:

```text
company-report.timer
```

activates:

```text
company-report.service
```

unless another unit is explicitly specified.

This naming convention makes relationships easy to discover.

List:

```bash
systemctl list-timers --all
```

Possible output:

```text
NEXT                        LEFT     LAST                        PASSED  UNIT                  ACTIVATES
Mon 2026-09-14 04:00:00...  7h      Sun 2026-09-13 04:00:...  16h     company-report.timer  company-report.service
```

This gives visibility cron does not provide natively:

```text
next activation
last activation
timer unit
activated service
```

---

## Creating a timer from scratch

Create:

```bash
sudo nano /etc/systemd/system/example-report.service
```

Content:

```ini
[Unit]
Description=Generate example report

[Service]
Type=oneshot
User=report
Group=report
WorkingDirectory=/srv/report
ExecStart=/usr/local/libexec/example-report
```

Create:

```bash
sudo nano /etc/systemd/system/example-report.timer
```

Content:

```ini
[Unit]
Description=Schedule example report

[Timer]
OnCalendar=*-*-* 03:15:00
Persistent=true

[Install]
WantedBy=timers.target
```

Reload unit definitions:

```bash
sudo systemctl daemon-reload
```

Enable and start timer:

```bash
sudo systemctl enable --now example-report.timer
```

Inspect:

```bash
systemctl status example-report.timer
```

List:

```bash
systemctl list-timers --all | grep example-report
```

Manually test the service without waiting:

```bash
sudo systemctl start example-report.service
```

Then:

```bash
systemctl status example-report.service
journalctl -u example-report.service
```

This workflow is one of systemd timers' strongest operational advantages.

The service can be tested independently of the schedule.

---

## Cron can also separate schedule and script

Cron does not force one-line complexity.

A clean cron architecture can be:

```cron
15 3 * * * /usr/local/libexec/example-report
```

with all execution logic inside:

```text
/usr/local/libexec/example-report
```

That can be excellent.

Therefore the comparison should not be:

```text
messy cron line
versus
clean systemd service
```

because cron can also be clean.

The real differences appear when system-level execution properties matter.

---

## Calendar syntax

Cron uses five primary scheduling fields:

```text
minute hour day-of-month month day-of-week
```

Example:

```cron
30 4 * * 1-5
```

means:

```text
04:30 Monday through Friday
```

Systemd timers can use calendar expressions such as:

```ini
OnCalendar=Mon..Fri *-*-* 04:30:00
```

or equivalent accepted syntax.

Test expressions with:

```bash
systemd-analyze calendar 'Mon..Fri *-*-* 04:30:00'
```

Example output may include:

```text
Original form: Mon..Fri *-*-* 04:30:00
Normalized form: Mon..Fri *-*-* 04:30:00
Next elapse: ...
From now: ...
```

This is extremely useful.

Cron has no universally available built-in command that explains the next matching time with the same richness.

---

## `systemd-analyze calendar`

Before installing a timer, test the calendar.

Example:

```bash
systemd-analyze calendar '*-*-* 02:00:00'
```

For every 15 minutes:

```bash
systemd-analyze calendar '*:0/15'
```

For Monday at 09:00:

```bash
systemd-analyze calendar 'Mon *-*-* 09:00:00'
```

For the first day of every month:

```bash
systemd-analyze calendar '*-*-01 00:00:00'
```

Always verify the expression on the target system.

Calendar grammar support can differ across systemd versions.

The local tool is authoritative.

---

## Monotonic timers

Cron is fundamentally calendar-based.

Systemd timers can also schedule relative to lifecycle events.

Examples include:

```ini
OnBootSec=10min
OnStartupSec=5min
OnUnitActiveSec=30min
OnUnitInactiveSec=10min
```

These are monotonic-style triggers.

Example:

```ini
[Timer]
OnBootSec=5min
OnUnitActiveSec=1h
```

means conceptually:

```text
first run five minutes after boot
then one hour after previous activation
```

This is different from cron:

```cron
0 * * * *
```

Cron runs on wall-clock hour boundaries.

The systemd timer runs relative to activation timing.

---

## Fixed wall-clock schedule versus interval schedule

Cron:

```cron
0 * * * * command
```

means:

```text
12:00
13:00
14:00
15:00
```

Systemd:

```ini
OnUnitActiveSec=1h
```

can behave more like:

```text
run
wait one hour
run again
```

If the previous run begins at:

```text
12:17
```

the next activation may be around:

```text
13:17
```

depending on unit behavior and timer configuration.

These are different scheduling models.

Choose intentionally.

---

## Drift

Suppose a job takes 10 minutes.

An interval defined as:

```text
one hour after activation
```

is not necessarily:

```text
one hour after completion
```

A timer using:

```ini
OnUnitInactiveSec=
```

can express:

```text
wait after service becomes inactive
```

more directly.

Example:

```ini
[Timer]
OnBootSec=5min
OnUnitInactiveSec=1h
```

Now:

```text
job completes
one hour passes
next run occurs
```

This may be a better model for periodic maintenance where overlap must never happen and spacing should be based on completion.

Cron cannot express this natively.

---

## Missed execution

Traditional cron may miss a job while the machine is down.

Anacron can add missed-period behavior.

Systemd timers can provide catch-up behavior with:

```ini
Persistent=true
```

Example:

```ini
[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
```

If the system is off at 02:00 and boots later, systemd can notice that the timer should have elapsed while inactive and trigger the service.

This provides anacron-like behavior while retaining calendar scheduling.

---

## `Persistent=true` is not a historical replay queue

Suppose a timer should run daily and the system is offline for five days.

Do not assume systemd will necessarily run the service five times to replay all missed dates.

Persistent timers generally catch up missed activation state rather than reconstruct every historical occurrence as separate business events.

If every missed business period matters, the application still needs durable backfill logic.

This is the same principle discussed with anacron.

Scheduler persistence is not business-event replay.

---

## Randomized delay

Systemd timers can introduce randomized delay:

```ini
RandomizedDelaySec=15min
```

Example:

```ini
[Timer]
OnCalendar=hourly
RandomizedDelaySec=10min
```

This spreads execution among many machines.

Useful for:

```text
package refresh
fleet health checks
telemetry
backup metadata
repository access
```

Without jitter, thousands of hosts scheduled at exactly:

```text
00 minutes
```

can overload a central service.

Cron can implement jitter in scripts, but systemd provides it as scheduler configuration.

---

## Accuracy windows

Systemd timers can coalesce wakeups using settings such as:

```ini
AccuracySec=
```

This allows the scheduler to run timers within an acceptable window rather than requiring precise second-level wakeups.

The goal can be:

```text
reduce wakeups
improve power efficiency
batch timers
```

For exact operational deadlines, choose values carefully.

Cron's model is simpler and usually minute-based.

---

## Seconds-level scheduling

Traditional cron usually has one-minute resolution.

A systemd timer can support finer timing.

Example:

```ini
OnUnitActiveSec=30s
```

for a 30-second interval.

Whether that is operationally appropriate is another question.

If a task needs execution every 5 seconds, a persistent service loop is usually better than repeatedly starting processes.

Scheduler capability does not imply good architecture.

---

## Dependency ordering

Cron does not natively understand:

```text
run after PostgreSQL is active
```

A script can check:

```bash
systemctl is-active --quiet postgresql
```

or retry.

Systemd can express:

```ini
After=postgresql.service
Wants=postgresql.service
```

or stronger relationships where appropriate.

Example:

```ini
[Unit]
Description=Database report
After=postgresql.service
Requires=postgresql.service
```

The exact dependency should match semantics.

`After=` controls ordering.

It does not automatically imply the dependency is started unless paired with relationships such as:

```text
Wants=
Requires=
```

Understanding that distinction is important.

---

## `After=` is not a health check

Suppose:

```ini
After=postgresql.service
```

Systemd starts the report after PostgreSQL's unit startup ordering.

That does not guarantee:

```text
database accepts queries
replication caught up
application schema migration finished
remote dependency healthy
```

The application should still perform relevant readiness checks.

Service ordering solves lifecycle ordering, not application health semantics.

---

## Mount dependencies

A backup job requires:

```text
/mnt/archive
```

Cron script:

```bash
mountpoint -q /mnt/archive || exit 1
```

That check is still useful.

Systemd can also express mount dependency:

```ini
RequiresMountsFor=/mnt/archive
```

This creates dependency relationships for mounts needed to access that path.

Service:

```ini
[Unit]
Description=Archive backup
RequiresMountsFor=/mnt/archive

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/archive-backup
```

The script should still validate business assumptions, but the service manager can handle system-level mount ordering.

---

## Network dependencies

A unit can use:

```ini
After=network-online.target
Wants=network-online.target
```

for workloads requiring configured network availability.

But:

```text
network-online.target
```

does not guarantee every remote endpoint is reachable.

VPN, DNS, remote API, and database readiness may still fail.

Again:

```text
system dependency
```

and:

```text
application readiness
```

are different layers.

---

## Working directory

Cron wrapper:

```bash
cd /srv/app || exit 1
exec /usr/bin/php artisan schedule:run
```

Systemd:

```ini
WorkingDirectory=/srv/app
ExecStart=/usr/bin/php artisan schedule:run
```

The systemd version exposes working-directory intent declaratively.

It also fails early if the directory does not exist, unless configured otherwise.

This can make diagnostics clearer.

---

## Environment variables

Cron:

```cron
APP_ENV=production
PATH=/usr/bin:/bin

0 2 * * * /opt/app/job
```

Systemd:

```ini
Environment=APP_ENV=production
Environment=PATH=/usr/bin:/bin
```

or:

```ini
EnvironmentFile=/etc/app/job.env
```

Be careful:

```text
EnvironmentFile=
```

has systemd's own parsing rules.

It is not identical to sourcing a shell script.

That can be a security advantage because arbitrary shell commands are not interpreted as shell code.

Still protect the file based on what values can influence execution.

---

## Secrets should not casually live in unit files

A service file such as:

```ini
Environment=DB_PASSWORD=secret
```

is not a good secret-management design.

Unit configuration can be inspected by administrators and tooling.

Use:

```text
protected credential files
systemd credentials where appropriate
secret manager
workload identity
application-native protected configuration
```

depending on systemd version and deployment environment.

Changing scheduler does not remove secret-management requirements.

---

## Service user

Cron system entry:

```cron
0 3 * * * report /opt/report/job
```

Systemd:

```ini
User=report
Group=report
```

Both can implement least privilege.

Systemd provides a richer security surface around that identity.

---

## `DynamicUser=`

For some ephemeral jobs, systemd can allocate a transient service identity with:

```ini
DynamicUser=yes
```

This can reduce the need for permanent service accounts.

It works best for services whose state/storage model is designed around systemd-managed directories.

This is an advanced option and should be tested carefully.

Cron has no direct equivalent.

---

## Runtime directories

Systemd can create runtime directories:

```ini
RuntimeDirectory=company-report
```

Then a path under:

```text
/run/
```

is managed for the service.

Likewise:

```ini
StateDirectory=
CacheDirectory=
LogsDirectory=
```

can create and manage service-specific directories on supported systemd versions.

This is valuable for avoiding manual:

```bash
mkdir
chown
chmod
```

boot logic.

---

## State directories

Example:

```ini
[Service]
Type=oneshot
User=report
StateDirectory=company-report
ExecStart=/usr/local/libexec/company-report
```

The application can use:

```text
/var/lib/company-report/
```

under systemd's managed directory model.

Exact paths and ownership behavior should be verified with:

```bash
man systemd.exec
```

for the installed version.

This is one example of systemd combining scheduler execution with service lifecycle.

---

## Timeout handling

Cron has no direct per-job timeout field.

A wrapper can use:

```bash
timeout 20m /opt/jobs/report
```

Systemd service:

```ini
TimeoutStartSec=20min
```

or related timeout settings depending on service type.

For a `Type=oneshot` workload, systemd can terminate a service that exceeds its configured start timeout.

This makes runtime policy part of the service unit rather than shell command syntax.

---

## Signal handling

When systemd stops a service, it has explicit signal and kill behavior controls.

Settings can influence:

```text
SIGTERM
SIGKILL escalation
process groups
child processes
kill mode
```

Cron does not manage the job as a long-lived service object after launch in the same way.

For complex process trees, systemd provides stronger lifecycle control.

---

## Child process tracking

A cron job can start:

```bash
script
    -> shell
        -> Python
            -> child
```

The scheduler may have little direct operational visibility into all descendants.

Systemd places service processes into a cgroup.

Inspect:

```bash
systemctl status company-report.service
```

while active.

Systemd can track processes belonging to the unit more reliably than process-name matching.

This is one of the strongest differences for production operations.

---

## Resource limits

Cron wrapper can use:

```bash
nice
ionice
ulimit
```

and system-wide cgroup configuration.

Systemd service can define resource policy declaratively.

Examples may include:

```ini
Nice=10
CPUWeight=20
MemoryMax=1G
TasksMax=100
```

depending on systemd/cgroup configuration.

This is useful for:

```text
backup
indexing
compression
security scans
large reports
```

that should not overwhelm production workloads.

---

## Memory limits

Example:

```ini
[Service]
Type=oneshot
User=report
ExecStart=/opt/report/run
MemoryMax=1G
```

If the process exceeds the cgroup memory policy, it can be terminated according to kernel/systemd behavior.

This creates a clear per-job resource boundary.

Cron alone has no native declarative memory limit.

---

## CPU controls

A heavy maintenance job can use:

```ini
Nice=10
CPUWeight=20
```

or other supported cgroup CPU controls.

This does not make the job faster.

It makes competition with more important workloads more controlled.

The exact effect depends on cgroup hierarchy and scheduler configuration.

---

## I/O controls

Systemd can expose cgroup I/O controls on supported systems.

This can be useful for a backup that otherwise saturates disks.

Traditional cron can call:

```bash
ionice
```

when applicable.

Systemd places resource policy next to the service definition, which improves discoverability.

---

## Sandboxing

A major systemd advantage is service sandboxing.

Possible settings include:

```ini
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=yes
PrivateDevices=yes
RestrictSUIDSGID=yes
LockPersonality=yes
```

Availability and semantics depend on systemd and kernel versions.

These settings can dramatically reduce the impact of a compromised scheduled program.

Cron itself does not provide comparable per-job sandbox declarations.

---

## `ProtectSystem=`

A service:

```ini
ProtectSystem=strict
```

can make much of the filesystem read-only from the service's namespace, with explicit writable paths granted as needed.

For example:

```ini
ReadWritePaths=/var/lib/company-report
```

This creates a strong boundary for a report generator that should not modify arbitrary system files.

A cron job running under the same UID relies primarily on ordinary filesystem permissions unless additional isolation is configured externally.

---

## `ProtectHome=`

If a system task has no reason to access user home directories:

```ini
ProtectHome=yes
```

can reduce exposure.

This is valuable for jobs processing:

```text
system logs
database backups
package metadata
```

that should not read:

```text
/home/alice
/home/bob
/root
```

Cron has no direct one-line equivalent.

---

## `PrivateTmp=`

With:

```ini
PrivateTmp=yes
```

the service receives private `/tmp` and `/var/tmp` namespaces.

This reduces interference with other users' temporary files.

It can also surprise applications that expect a file written by another service under:

```text
/tmp
```

to be visible.

Security controls must match application behavior.

---

## `NoNewPrivileges=`

```ini
NoNewPrivileges=yes
```

prevents the service and descendants from gaining additional privileges through certain mechanisms such as setuid execution.

This is useful when the service should remain at its configured privilege level.

It is not a replacement for:

```ini
User=
```

If the service already starts as root, `NoNewPrivileges=yes` does not turn root into an unprivileged account.

---

## Capability controls

Systemd can restrict Linux capabilities:

```ini
CapabilityBoundingSet=
AmbientCapabilities=
```

A task that needs one privileged operation may run without full root authority.

Example:

```text
network-related capability
```

instead of full root, when supported by the application.

Cron can also execute capability-enabled binaries, but systemd provides a direct unit-level policy language.

---

## System call filtering

Systemd can use settings such as:

```ini
SystemCallFilter=
```

to limit permitted system calls.

This is advanced sandboxing.

It can substantially reduce attack surface, but incorrect filters can break applications in non-obvious ways.

Use:

```bash
systemd-analyze security UNIT
```

as one aid, not as an automatic policy generator.

---

## `systemd-analyze security`

Run:

```bash
systemd-analyze security company-report.service
```

The output evaluates many hardening properties and gives an exposure score.

This is useful for reviewing:

```text
scheduled services
daemons
one-shot maintenance tasks
```

The score is not proof of security.

It highlights hardening opportunities.

A service may legitimately require capabilities that increase its score.

Interpret results in application context.

---

## Logging

Cron jobs often require:

```bash
>> /var/log/job.log 2>&1
```

or `logger`.

Systemd services integrate naturally with journald.

Example:

```ini
StandardOutput=journal
StandardError=journal
```

Often this is already the default for service output.

Inspect:

```bash
journalctl -u company-report.service
```

Follow:

```bash
journalctl -fu company-report.service
```

Filter current boot:

```bash
journalctl -u company-report.service -b
```

This gives per-unit log grouping automatically.

---

## Exit status visibility

After a one-shot service fails:

```bash
systemctl status company-report.service
```

can show:

```text
Process: ... ExecStart=...
Main PID: ...
status=...
Result: exit-code
```

Detailed logs:

```bash
journalctl -u company-report.service
```

Cron does not provide a comparable native per-job status object.

With cron, you typically need:

```text
wrapper logs
mail
state files
monitoring
```

Systemd service state makes failure inspection easier.

---

## Last result

Use:

```bash
systemctl show company-report.service
```

Relevant properties may include:

```text
ActiveState
SubState
Result
ExecMainStatus
ExecMainCode
```

Exact property meaning depends on service state and version.

This gives machine-readable service execution information.

Cron does not maintain equivalent standardized unit metadata per scheduled command.

---

## Manual execution

Cron job testing:

```bash
sudo -u report /opt/report/job
```

possibly with minimal environment.

Systemd job testing:

```bash
sudo systemctl start company-report.service
```

Then:

```bash
systemctl status company-report.service
journalctl -u company-report.service
```

This tests the same:

```text
User=
WorkingDirectory=
Environment=
sandbox
limits
```

that the timer will use.

That is operationally powerful.

---

## Timer testing without waiting

Test service directly.

Then inspect timer schedule:

```bash
systemd-analyze calendar 'daily'
```

or exact expression.

List next run:

```bash
systemctl list-timers company-report.timer
```

This separates:

```text
service correctness
timer correctness
```

just as good cron practice separates:

```text
script correctness
schedule correctness
```

---

## Enabling a timer

After creating units:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now company-report.timer
```

`enable` generally creates boot-time activation links according to:

```ini
[Install]
WantedBy=timers.target
```

`--now` also starts the timer immediately.

Without `--now`, enabling affects future activation after boot or explicit start but does not necessarily start the current timer instance immediately.

---

## Starting versus enabling

```bash
systemctl start example.timer
```

starts now but does not necessarily enable at boot.

```bash
systemctl enable example.timer
```

enables for future boot but does not necessarily start it immediately.

```bash
systemctl enable --now example.timer
```

does both.

This distinction is important when migrating from cron.

---

## Disabling a timer

```bash
sudo systemctl disable --now example.timer
```

stops current timer and removes enablement links.

The service unit still exists.

An administrator can still manually run:

```bash
sudo systemctl start example.service
```

This separation between:

```text
schedule enabled
workload available
```

is useful during maintenance.

---

## Masking

To prevent a unit from being started at all:

```bash
sudo systemctl mask example.service
```

A mask is stronger than disable.

It links the unit to a null target and prevents normal activation.

Use carefully.

Unmask:

```bash
sudo systemctl unmask example.service
```

Masking is not a routine scheduling control.

It is useful when a unit must be prevented from running.

---

## Timer active while service is active

One important operational behavior is that a timer activating the same service unit does not simply create arbitrary duplicate copies of that same service while it is already active.

Systemd unit state provides natural serialization for many timer/service patterns.

This differs from cron, where:

```text
every matching time
```

can launch another process unless the job implements locking.

However, exact behavior depends on service type and unit state.

Do not assume this solves every application-level concurrency problem.

---

## Long-running `Type=oneshot` service

A `Type=oneshot` service stays activating until its command exits.

If the timer elapses again while the service is still active, systemd's unit activation semantics generally avoid creating another independent instance of the same non-template service.

That makes many simple timer workloads naturally non-overlapping.

This is a strong advantage over cron.

Still monitor long runtimes.

A timer silently "not overlapping" can hide a permanently stuck service if you only look at duplicate process count.

---

## Do you still need locking with systemd timers?

Sometimes no.

If:

```text
only one timer on one host
same service unit
manual runs always use systemctl
```

unit state can provide enough local serialization.

But locking may still be needed if:

```text
same script can run outside systemd
multiple different units access same resource
multiple hosts run same workflow
application itself has parallel workers
external scheduler can invoke same operation
```

Concurrency belongs to the shared resource, not the timer technology.

---

## Multiple timers can activate one service

A service can be activated by more than one timer.

Example:

```text
daily-report.timer
report-on-boot.timer
```

both activate:

```text
report.service
```

This can be intentional.

It also means service-level idempotency remains valuable.

Discover relationships with:

```bash
systemctl list-dependencies report.service
```

and unit inspection.

---

## Template services

Systemd template units such as:

```text
worker@.service
```

can create multiple instances:

```text
worker@a.service
worker@b.service
```

These are independent units.

Unit-level non-overlap applies per instance, not globally across all template instances.

If all instances share a database resource, application-level coordination may still be required.

---

## OnFailure

Systemd services can trigger another unit when they fail using relationships such as:

```ini
OnFailure=company-report-failure.service
```

This can support:

```text
alerting
incident hooks
cleanup
```

Use carefully to avoid recursive failure chains.

Cron usually requires error-handling logic inside scripts or external monitoring.

---

## Success exit status customization

Some programs use non-zero exit statuses for acceptable conditions.

Systemd can customize success interpretation with service settings such as:

```ini
SuccessExitStatus=
```

when appropriate.

For example, if an application documents:

```text
exit 2 = nothing to do
```

the service can potentially treat that as successful.

Do not hide real failures by broadly accepting many codes.

Use application semantics intentionally.

---

## Restart behavior

A scheduled one-shot job usually should not restart forever.

But a service can define:

```ini
Restart=on-failure
RestartSec=30s
```

depending on type and workload.

This gives systemd retry-like behavior.

However, repeated restart of a one-shot task can create:

```text
duplicate external effects
load storms
unbounded retries
```

Use retry policy only when the job is safe to repeat.

---

## Start limits

Systemd can rate-limit repeated activations/restarts through unit start-limit settings.

This protects against rapid failure loops.

Cron provides no native comparable service activation rate limiter beyond its schedule.

Again, richer lifecycle comes with richer configuration.

---

## Resource cleanup

A systemd service's cgroup gives the manager a clear set of related processes.

When the service is stopped or times out, systemd can terminate descendants according to configured kill behavior.

A cron shell script that backgrounds children can leave them behind.

This is one reason backgrounding inside cron jobs is usually a poor design.

---

## Avoid `nohup ... &` under systemd too

A service:

```ini
ExecStart=/bin/sh -c 'nohup /opt/app/job &'
```

defeats much of systemd's process supervision.

Let systemd own the foreground process.

Example:

```ini
ExecStart=/opt/app/job
```

If the application daemonizes itself, configure service type appropriately.

For scheduled one-shot tasks, remain foreground.

---

## Security boundary example

Cron:

```cron
0 2 * * * report /opt/report/run
```

The account `report` may have access to:

```text
all files readable by report
all home paths allowed by DAC
all network destinations allowed by host policy
```

Systemd:

```ini
[Service]
Type=oneshot
User=report
ExecStart=/opt/report/run
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
ReadWritePaths=/var/lib/report
```

Now even if `report` normally has broader access, the service can run inside a narrower execution sandbox.

This can significantly reduce blast radius.

---

## Security is not automatic

A systemd unit:

```ini
[Service]
User=root
ExecStart=/home/alice/job.sh
```

is still insecure if Alice can modify the script.

Systemd timers do not fix:

```text
writable privileged code
unsafe interpreter path
untrusted config
command injection
weak credentials
```

The same trust-chain principles apply.

---

## Root versus non-root user units

Systemd also supports per-user units.

A user can have timers under locations such as:

```text
~/.config/systemd/user/
```

depending on environment.

Commands:

```bash
systemctl --user daemon-reload
systemctl --user enable --now example.timer
systemctl --user list-timers
```

User timers can replace personal crontabs for some desktop/server user workflows.

Their lifecycle may depend on user manager configuration and lingering behavior.

---

## User lingering

A user systemd manager may stop after logout unless lingering or other session behavior keeps it available.

Inspect:

```bash
loginctl show-user USER
```

Enable lingering when appropriate:

```bash
sudo loginctl enable-linger USER
```

This allows a user manager to remain active without an interactive login on many systemd systems.

This is a security and operational decision.

Do not enable it mechanically for every account.

---

## Personal crontab remains simpler

For a user's harmless daily script:

```cron
0 9 * * * /home/alice/bin/report
```

a crontab may be far easier than:

```text
user service
user timer
linger
daemon-reload
enablement
```

Systemd's power should not become unnecessary complexity.

Use the simplest mechanism that satisfies the requirements.

---

## Portability

Cron exists across:

```text
Linux
BSD
commercial Unix
embedded Unix-like systems
many containers
```

Systemd timers exist only where systemd is the service manager.

If a project must deploy across heterogeneous Unix systems, cron is more portable.

If infrastructure is explicitly:

```text
modern Linux with systemd
```

portability may not matter.

---

## Configuration familiarity

Most administrators immediately understand:

```cron
0 2 * * *
```

Fewer people may be comfortable debugging:

```text
timer dependencies
service state
unit precedence
drop-ins
calendar syntax
cgroup policy
```

Operational team skill matters.

A theoretically powerful scheduler is not automatically better if nobody can operate it safely.

Documentation and standardization reduce this problem.

---

## Configuration size

Cron job:

```cron
15 2 * * * /usr/local/sbin/backup
```

One line.

Systemd:

```text
backup.service
backup.timer
```

often 15-30 lines combined.

For ten trivial tasks, systemd can create significant configuration volume.

That verbosity is justified when it expresses useful behavior.

It is wasteful when every job is simply:

```text
run one harmless command once a day
```

---

## Discoverability

Cron schedules can be spread across:

```text
user crontabs
root crontab
/etc/crontab
/etc/cron.d/
/etc/cron.daily/
```

Systemd timers can be listed centrally:

```bash
systemctl list-timers --all
```

User timers require:

```bash
systemctl --user list-timers --all
```

in relevant user managers.

Neither ecosystem is perfectly centralized, but systemd provides stronger runtime discovery.

---

## Finding all cron schedules

Classic audit:

```bash
crontab -l
sudo crontab -l
sudo cat /etc/crontab
sudo grep -R -n -v '^[[:space:]]*#' /etc/cron.d 2>/dev/null
run-parts --test /etc/cron.hourly
run-parts --test /etc/cron.daily
run-parts --test /etc/cron.weekly
run-parts --test /etc/cron.monthly
```

This is more fragmented than:

```bash
systemctl list-timers --all
```

but cron remains transparent because everything is ordinary text and executable files.

---

## Inspecting unit definitions

Show active merged unit content:

```bash
systemctl cat company-report.service
```

and:

```bash
systemctl cat company-report.timer
```

This includes drop-ins.

That is better than reading only:

```text
/etc/systemd/system/company-report.service
```

because package units and overrides can be merged from several paths.

---

## Unit search paths

Systemd units can come from locations such as:

```text
/etc/systemd/system/
/run/systemd/system/
/usr/lib/systemd/system/
```

or:

```text
/lib/systemd/system/
```

depending on distribution.

Local administrator overrides normally belong under:

```text
/etc/systemd/system/
```

Package-owned units usually live under vendor paths.

Do not directly edit vendor unit files unless you intentionally want package upgrades to overwrite changes.

---

## Drop-in overrides

Use:

```bash
sudo systemctl edit company-report.service
```

This can create a drop-in under:

```text
/etc/systemd/system/company-report.service.d/
```

Example:

```ini
[Service]
MemoryMax=512M
```

The package-owned base unit remains untouched.

This is a strong configuration-management pattern.

Cron's `/etc/cron.d/` also provides drop-in style separation, but not per-entry layered override semantics.

---

## Verify units

Systemd can validate unit files:

```bash
systemd-analyze verify \
    /etc/systemd/system/company-report.service \
    /etc/systemd/system/company-report.timer
```

This catches some syntax and dependency issues.

It does not prove application correctness.

Still test:

```bash
systemctl start company-report.service
```

---

## `daemon-reload`

After adding or modifying unit files:

```bash
sudo systemctl daemon-reload
```

is normally required so systemd rereads unit definitions.

Cron often notices crontab file changes automatically.

This is a procedural difference administrators must remember.

---

## Forgetting `daemon-reload`

You edit:

```text
company-report.service
```

then:

```bash
systemctl start company-report.service
```

and old behavior persists.

Systemd may warn:

```text
unit file changed on disk
run daemon-reload
```

The fix:

```bash
sudo systemctl daemon-reload
```

This class of stale-configuration problem does not usually exist in the same form with cron.

---

## Cron reload behavior

Cron implementations commonly detect crontab changes automatically.

But exact reload mechanics vary.

You generally do not need:

```bash
systemctl restart cron
```

after every `crontab -e`.

That simplicity is valuable.

Do not restart cron unnecessarily.

---

## Boot execution

Cron daemon normally starts as a service.

`@reboot` can schedule:

```cron
@reboot /opt/jobs/startup-task
```

But `@reboot` means roughly:

```text
when cron daemon starts
```

not:

```text
when all system dependencies are ready
```

Systemd units are stronger for boot-dependent ordering.

Example:

```ini
After=network-online.target
After=postgresql.service
```

with:

```ini
[Timer]
OnBootSec=10min
```

This makes startup timing more explicit.

---

## `@reboot` versus `OnBootSec`

Cron:

```cron
@reboot /opt/jobs/task
```

Systemd timer:

```ini
OnBootSec=10min
```

The systemd version explicitly waits ten minutes after boot.

Cron wrapper could emulate:

```cron
@reboot sleep 600 && /opt/jobs/task
```

but this is less declarative and lifecycle-aware.

---

## Shell usage

Cron command lines are interpreted through a shell in traditional implementations.

Systemd `ExecStart=` is not a shell command line by default.

This is important.

This will not behave like a shell pipeline:

```ini
ExecStart=/usr/bin/cat /var/log/app.log | /usr/bin/gzip
```

Systemd does not automatically interpret:

```text
|
>
&&
*
$VARIABLE
```

through `/bin/sh`.

That is a feature.

It avoids accidental shell parsing.

---

## If systemd needs a shell

Use an explicit shell:

```ini
ExecStart=/bin/sh -c '/usr/bin/cat /var/log/app.log | /usr/bin/gzip > /var/backups/app.log.gz'
```

But if command logic becomes complex, create a script:

```text
/usr/local/libexec/compress-log
```

and:

```ini
ExecStart=/usr/local/libexec/compress-log
```

This mirrors best practice with cron.

Keep scheduler/service configuration readable.

---

## Environment expansion is different

Do not copy a cron shell command into `ExecStart=` and assume identical variable expansion.

Systemd has its own command parsing and environment substitution rules.

Always check:

```bash
man systemd.service
man systemd.exec
```

for the local version.

This is a frequent migration error.

---

## Redirection migration

Cron:

```cron
0 2 * * * /opt/job >> /var/log/job.log 2>&1
```

Systemd should not be translated literally as:

```ini
ExecStart=/opt/job >> /var/log/job.log 2>&1
```

because `ExecStart=` is not normally parsed by a shell.

Use journald:

```ini
StandardOutput=journal
StandardError=journal
```

or supported file-output directives where appropriate.

Journald is usually the simplest operational default.

---

## Cron `PATH` versus systemd path

A cron job often has a restricted `PATH`.

Systemd services also run with a controlled environment.

Do not rely on an interactive shell path in either system.

Use explicit executable paths:

```ini
ExecStart=/usr/bin/python3 /opt/app/job.py
```

rather than:

```ini
ExecStart=python3 /opt/app/job.py
```

unless path resolution is intentionally configured.

---

## Cron `%` escaping disappears in service units

Cron has special `%` behavior in command fields on traditional implementations.

A command like:

```cron
date +\%F
```

requires cron-specific escaping.

A systemd service invoking:

```ini
ExecStart=/usr/bin/date +%F
```

does not use crontab's `%` parser.

However, systemd has its own `%` specifier syntax in unit files.

Literal percent handling can therefore require different escaping depending on context and directive.

Do not mechanically translate escaping rules between scheduler formats.

---

## systemd specifiers

Systemd uses `%` specifiers such as:

```text
%n
%i
%u
```

in many directives.

Therefore percent signs can still have special meaning, just for a different reason.

Consult:

```bash
man systemd.unit
```

and the relevant directive documentation.

Migration must account for parser differences.

---

## Failure notification

Cron traditionally relies on:

```text
mail
logs
wrapper scripts
monitoring
```

Systemd can integrate:

```text
journal
OnFailure
service result
monitoring that queries unit state
```

Neither automatically gives complete enterprise alerting.

You still need:

```text
Prometheus
SIEM
alert manager
health-check service
email integration
```

or another monitoring path.

---

## Monitoring a timer is not enough

A timer can be active:

```bash
systemctl is-active company-report.timer
```

while the service has been failing for days.

Monitor:

```text
timer active
last service result
last successful application completion
business artifact freshness
```

The same lesson applies to cron:

```text
scheduler alive
```

does not imply:

```text
job successful
```

---

## Checking failed units

List failures:

```bash
systemctl --failed
```

A failed one-shot service can remain visible as failed.

This is useful because a cron job that failed at 03:00 may leave no standardized scheduler state by 09:00.

Systemd preserves a unit failure state until reset or superseded.

---

## Resetting failure state

After fixing/testing:

```bash
sudo systemctl reset-failed company-report.service
```

This clears failed state metadata.

It does not fix the application.

Use it after understanding and correcting the problem.

---

## One-shot service state after success

A `Type=oneshot` service without:

```ini
RemainAfterExit=yes
```

normally becomes inactive after successful command completion.

That is expected.

The timer remains active and schedules the next run.

Do not interpret:

```text
service inactive
```

as failure.

Check:

```bash
systemctl status service
journalctl
timer state
```

in context.

---

## `RemainAfterExit=yes` can break repeated timer behavior

If a one-shot service uses:

```ini
RemainAfterExit=yes
```

it can remain active after the command exits.

A timer later trying to activate an already active service may not rerun the command as expected.

For recurring timer jobs, `RemainAfterExit=yes` is often inappropriate.

Use it only when the service semantics genuinely require the unit to remain active.

This is a subtle but important timer design issue.

---

## Persistent timer state

Systemd tracks timer state under its own manager state.

Administrators generally should not manually edit internal timestamp files.

Inspect with supported interfaces:

```bash
systemctl list-timers --all
systemctl show timer
```

rather than manipulating implementation files.

The same principle applies to cron spool internals and anacron state.

Use scheduler APIs.

---

## Real-time versus monotonic clocks

`OnCalendar=` uses wall-clock/calendar concepts.

Monotonic timers such as:

```ini
OnBootSec=
OnUnitActiveSec=
```

are based on monotonic elapsed-time concepts.

Wall-clock changes affect them differently.

This distinction matters for:

```text
NTP corrections
manual clock changes
timezone changes
DST
suspend
```

Systemd documentation describes which timer bases advance during suspend and which do not.

Check the target version for precise semantics.

---

## `WakeSystem=`

Some systemd timer configurations can request wake-from-suspend behavior with:

```ini
WakeSystem=true
```

on supported hardware/configurations.

This is far outside traditional cron's scope.

It may be useful for appliances or laptops that must wake for maintenance.

Hardware, firmware, kernel, and power policy all influence whether this works.

Test on the target machine.

---

## Calendar timezone

Systemd calendar expressions can support timezone-aware syntax on versions that provide that functionality.

Cron traditionally follows system/daemon timezone with implementation-specific features such as `CRON_TZ` on some versions.

For global systems, do not rely on vague "daily" semantics.

Explicitly document:

```text
UTC
server local time
customer timezone
business timezone
```

regardless of scheduler.

---

## Daylight-saving transitions

Cron and systemd both must interpret local calendar schedules through timezone transitions.

A local time can:

```text
not exist
occur twice
```

during DST changes.

Systemd's calendar tooling makes upcoming activations easier to inspect.

For business-critical daily processes, application-level idempotency remains the best protection against duplicate or skipped calendar effects.

---

## Cron's main strengths

Cron is excellent when:

```text
schedule is simple
job is short
machine is usually online
execution environment is straightforward
shell wrapper already handles logging/locking
portability matters
team understands cron
```

Examples:

```text
rotate a local generated file
run a short cleanup
call a monitoring probe
refresh a cache
run a simple report
```

Its small surface area is a strength.

---

## Systemd timer strengths

Systemd timers are excellent when:

```text
job has service dependencies
missed-run catch-up matters
resource limits matter
sandboxing matters
unit-level logging matters
boot ordering matters
timeout policy matters
Linux/systemd is guaranteed
```

Examples:

```text
database maintenance
security-sensitive root task
resource-heavy backup
job requiring mounted storage
job requiring network-online ordering
long-running index rebuild
```

---

## Cron weaknesses

Cron lacks native per-job abstractions for:

```text
cgroup resource control
systemd dependency graph
service sandbox
runtime timeout
next-run inspection
unit status
persistent catch-up
monotonic interval scheduling
```

These features can be implemented around cron, but not as part of cron itself.

---

## Systemd timer weaknesses

Systemd timers have costs:

```text
more configuration
Linux/systemd dependency
steeper learning curve
more concepts
unit parsing rules
daemon-reload workflow
```

For a trivial personal job, that complexity can be unnecessary.

---

## Do not choose based on fashion

This is poor reasoning:

```text
cron is old, therefore bad
```

and:

```text
systemd is complicated, therefore bad
```

A better decision is:

```text
what failure modes matter for this workload?
```

Then choose the smallest tool that controls those failure modes.

---

## Example: simple cache cleanup

Requirement:

```text
delete local cache files older than 14 days
run once every night
exact time not critical
host always online
```

Cron is excellent:

```cron
20 3 * * * /usr/local/sbin/clean-cache
```

Script:

```bash
#!/bin/sh
set -eu

exec /usr/bin/find \
    /var/cache/company \
    -type f \
    -mtime +14 \
    -delete
```

Systemd would work too.

It probably adds little value unless sandboxing or missed-run behavior is needed.

---

## Example: local backup requiring a mount

Requirement:

```text
daily
catch up after reboot
destination mount required
limit runtime
log centrally
no home-directory access
```

Systemd becomes attractive.

Service:

```ini
[Unit]
Description=Company backup
RequiresMountsFor=/mnt/backup

[Service]
Type=oneshot
User=backup
Group=backup
ExecStart=/usr/local/libexec/company-backup
TimeoutStartSec=2h
ProtectHome=yes
PrivateTmp=yes
NoNewPrivileges=yes
```

Timer:

```ini
[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
RandomizedDelaySec=10min

[Install]
WantedBy=timers.target
```

This expresses operational requirements directly.

---

## Example: Laravel scheduler

A common Laravel host uses:

```cron
* * * * * cd /srv/app && /usr/bin/php artisan schedule:run >> /dev/null 2>&1
```

This is simple and widely understood.

A systemd alternative:

```ini
[Service]
Type=oneshot
User=www-data
Group=www-data
WorkingDirectory=/srv/app
ExecStart=/usr/bin/php artisan schedule:run
```

Timer:

```ini
[Timer]
OnCalendar=*-*-* *:*:00
AccuracySec=1s

[Install]
WantedBy=timers.target
```

Which is better depends on deployment.

If the application framework already handles:

```text
overlap
single-server execution
task logging
```

cron may be perfectly adequate.

If the platform wants:

```text
unit logs
security sandbox
resource policy
service-account lifecycle
```

systemd is attractive.

---

## Example: Docker Compose maintenance

Cron:

```cron
0 4 * * * /usr/bin/docker compose -f /srv/app/compose.yml exec -T app php artisan cleanup
```

Systemd service:

```ini
[Service]
Type=oneshot
WorkingDirectory=/srv/app
ExecStart=/usr/bin/docker compose -f /srv/app/compose.yml exec -T app php artisan cleanup
```

Timer:

```ini
[Timer]
OnCalendar=*-*-* 04:00:00
Persistent=true
```

The systemd version gives better logging/lifecycle visibility.

But the service account still has Docker daemon authority.

Scheduler migration does not reduce that privilege automatically.

---

## Example: database backup

Cron:

```cron
0 1 * * * postgres /usr/local/libexec/db-backup
```

This may be enough if:

```text
server always online
script validates output
locking implemented
monitoring exists
```

Systemd service:

```ini
[Unit]
Description=PostgreSQL backup
After=postgresql.service
Requires=postgresql.service

[Service]
Type=oneshot
User=postgres
ExecStart=/usr/local/libexec/db-backup
TimeoutStartSec=3h
Nice=10
```

Timer:

```ini
[Timer]
OnCalendar=*-*-* 01:00:00
Persistent=true
RandomizedDelaySec=5min

[Install]
WantedBy=timers.target
```

The systemd model is richer if operational controls matter.

---

## Example: security scan

A root security scan may benefit strongly from systemd sandboxing.

Service:

```ini
[Service]
Type=oneshot
User=scanner
Group=scanner
ExecStart=/usr/local/bin/security-scan
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=/var/lib/security-scan
```

Timer:

```ini
[Timer]
OnCalendar=weekly
Persistent=true
RandomizedDelaySec=2h
```

This is a strong use case for systemd.

---

## Example: personal script

User wants:

```text
run ~/bin/weather-cache at 08:00
```

Crontab:

```cron
0 8 * * * /home/alice/bin/weather-cache
```

This is likely simpler than creating user units.

The execution has low operational risk.

Use cron.

---

## Example: job after a service finishes

Requirement:

```text
run cleanup 10 minutes after backup service becomes inactive
```

Systemd can express:

```ini
OnUnitInactiveSec=10min
Unit=backup-cleanup.service
```

depending on architecture.

Cron cannot naturally schedule relative to service completion.

That is a clear systemd advantage.

---

## Example: run every 30 minutes after completion

Service runtime:

```text
variable
```

Requirement:

```text
wait 30 minutes after the previous run finishes
```

Systemd timer:

```ini
OnUnitInactiveSec=30min
```

is naturally suited.

Cron:

```cron
*/30 * * * *
```

does not mean the same thing.

If a run takes 20 minutes:

```text
cron gives only 10 minutes before next scheduled start
```

Systemd can model the intended interval more accurately.

---

## Example: exact wall-clock report

Requirement:

```text
09:00 every business day
```

Cron:

```cron
0 9 * * 1-5 /opt/report
```

is excellent.

Systemd:

```ini
OnCalendar=Mon..Fri *-*-* 09:00:00
```

is also excellent.

Choose based on execution-management needs.

The schedule itself does not decide the winner.

---

## Example: server may be offline

Requirement:

```text
daily report
server may be powered off overnight
run as soon as practical after return
```

Options:

```text
anacron
systemd timer with Persistent=true
application catch-up
```

Traditional cron alone is weak.

On a modern systemd server, a persistent timer is often the cleanest.

On a traditional Unix environment, anacron may be more portable.

---

## Migration from cron to systemd

Suppose:

```cron
0 2 * * * backup /usr/local/libexec/company-backup >> /var/log/company-backup.log 2>&1
```

First document current semantics:

```text
user: backup
time: 02:00
working directory: cron default
environment: cron environment
output: file
missed runs: no catch-up
overlap: possible unless script locks
timeout: none
```

Do not migrate only the visible command.

Migrate behavior.

---

## Migration service

```ini
[Unit]
Description=Company backup

[Service]
Type=oneshot
User=backup
Group=backup
ExecStart=/usr/local/libexec/company-backup
StandardOutput=journal
StandardError=journal
```

Timer:

```ini
[Unit]
Description=Schedule company backup

[Timer]
OnCalendar=*-*-* 02:00:00

[Install]
WantedBy=timers.target
```

Notice:

```text
Persistent=true
```

is not included yet because the original cron job did not catch up missed runs.

Adding it would change semantics.

Migration should preserve behavior first unless the change is intentional.

---

## Validate migration

Check units:

```bash
sudo systemd-analyze verify \
    /etc/systemd/system/company-backup.service \
    /etc/systemd/system/company-backup.timer
```

Reload:

```bash
sudo systemctl daemon-reload
```

Test service:

```bash
sudo systemctl start company-backup.service
```

Inspect:

```bash
systemctl status company-backup.service
journalctl -u company-backup.service
```

Test calendar:

```bash
systemd-analyze calendar '*-*-* 02:00:00'
```

Enable timer:

```bash
sudo systemctl enable --now company-backup.timer
```

Inspect:

```bash
systemctl list-timers company-backup.timer
```

Only after confirming the new timer should the old cron schedule be removed.

---

## Avoid duplicate scheduling during migration

If cron and systemd timer are both active:

```text
02:00 cron launches backup
02:00 systemd launches backup
```

Two copies may run.

Even if the backup script locks, one run may skip and monitoring may become confusing.

Search:

```bash
sudo grep -R -n 'company-backup' /etc/cron* 2>/dev/null
sudo crontab -l
systemctl list-timers --all | grep company-backup
```

Migration is complete only when one scheduler owns the task.

---

## Rollback plan

Before removing cron:

```bash
crontab -l > /root/crontab.backup
```

or copy relevant system cron file.

For `/etc/cron.d/`:

```bash
sudo cp -a \
    /etc/cron.d/company-backup \
    /root/company-backup.cron.backup
```

After timer validation, remove or disable old schedule.

If the timer fails unexpectedly, rollback can restore the original scheduler quickly.

Configuration changes should be reversible.

---

## Migration from systemd back to cron

Sometimes portability matters later.

Before converting:

```text
inspect User=
inspect WorkingDirectory=
inspect Environment=
inspect resource limits
inspect dependencies
inspect timeout
inspect sandbox settings
inspect Persistent=
inspect randomized delay
```

A one-line cron conversion may silently lose critical behavior.

Example:

```ini
ProtectSystem=strict
MemoryMax=1G
TimeoutStartSec=20min
RequiresMountsFor=/mnt/backup
```

has no direct cron fields.

You would need wrappers or external system configuration.

Do not claim semantic equivalence unless those controls are recreated.

---

## Real-world operations: deployment

Scheduled tasks should be deployed like application code.

For cron:

```text
install script
install cron definition
set owner/mode
validate syntax
test script
observe next run
```

For systemd:

```text
install service
install timer
daemon-reload
verify
test service
enable timer
observe next run
```

Both deserve configuration management.

---

## Use Git or configuration management

Store:

```text
cron definitions
systemd units
scripts
logrotate config
monitoring rules
```

in version control.

Do not treat:

```text
crontab -e
```

as an undocumented production database.

The same is true of manually edited:

```text
/etc/systemd/system/
```

Operational reproducibility matters more than scheduler choice.

---

## Rollout across many servers

A fleet deployment should control:

```text
which hosts receive schedule
which user executes it
whether jobs are staggered
whether one host is designated leader
whether all hosts may run independently
```

Configuration management can select:

```text
role=scheduler
```

and install the unit only there.

Scheduler duplication is a deployment problem as much as a cron/systemd problem.

---

## Host role example

Inventory:

```text
web-01 role=web
web-02 role=web
worker-01 role=worker
scheduler-01 role=scheduler
```

Only:

```text
scheduler-01
```

receives:

```text
billing.timer
report.timer
```

This avoids accidental multi-host duplication.

The same pattern works with cron.

---

## High availability

A designated scheduler host is simple but creates a single point of scheduling failure.

Options for higher availability:

```text
distributed job queue
database advisory locks
cluster-aware scheduler
Kubernetes CronJob
cloud scheduler
active/passive leader election
```

Systemd timer and cron are both fundamentally local schedulers.

Neither alone solves cluster leadership.

---

## Monitoring cron in production

Useful signals:

```text
cron daemon active
job last start
job last success
job last failure
duration
output freshness
artifact validation
lock skips
```

A scheduler health check should not stop at:

```bash
systemctl is-active cron
```

That proves only the daemon.

---

## Monitoring systemd timers

Useful:

```bash
systemctl list-timers --all
```

Check failed services:

```bash
systemctl --failed
```

Inspect one unit:

```bash
systemctl show company-report.service \
    -p Result \
    -p ExecMainStatus \
    -p ActiveState \
    -p SubState
```

But still monitor application success.

A service can exit `0` while producing bad data.

---

## Artifact freshness remains scheduler-independent

Backup monitoring:

```bash
find /var/backups/app \
    -type f \
    -name '*.tar.gz' \
    -mmin -1560 \
    -print \
    -quit
```

Report monitoring:

```bash
stat /var/lib/report/last-success
```

Database workflow:

```text
last completed business period
```

These checks remain useful whether the trigger is:

```text
cron
anacron
systemd
Kubernetes
cloud scheduler
```

Business outcome monitoring should not depend too strongly on scheduler technology.

---

## Logging policy should survive migration

If cron used:

```text
/var/log/company-report.log
```

and systemd migration switches to journald, update:

```text
runbooks
monitoring
log forwarding
retention
alert rules
```

Otherwise operators may look in the wrong place.

A scheduler migration is also an observability migration.

---

## Journald retention

Journald has its own storage and retention configuration.

Inspect:

```bash
journalctl --disk-usage
```

Configuration may live under:

```text
/etc/systemd/journald.conf
/etc/systemd/journald.conf.d/
```

Do not assume logs remain forever.

If audit requirements require 90 days, configure retention or forward logs externally.

Systemd logging integration does not remove retention planning.

---

## File logs may still be appropriate

Some applications already manage:

```text
structured JSON logs
rotation
central shipping
```

There is no rule that systemd services must log only to journald.

Choose one consistent logging architecture.

Avoid duplicating huge output into:

```text
journald
and
application file
```

without reason.

---

## Real-world debugging: cron version

Problem:

```text
daily report missing
```

Workflow:

```bash
systemctl status cron
journalctl -u cron --since "04:00"
sudo -u report /usr/local/libexec/report
```

Then:

```bash
namei -l /usr/local/libexec/report
```

and application logs.

The administrator must reconstruct execution state manually.

---

## Real-world debugging: systemd timer version

Workflow:

```bash
systemctl status report.timer
systemctl status report.service
journalctl -u report.service
systemctl show report.service \
    -p Result \
    -p ExecMainStatus
```

Then run:

```bash
sudo systemctl start report.service
```

This is often faster operationally.

That is one of systemd's strongest advantages for important system tasks.

---

## Failure example: wrong working directory

Cron:

```cron
0 5 * * * /srv/app/bin/report
```

Script depends on:

```text
./config.yaml
```

Fails.

Fix cron wrapper or script.

Systemd:

```ini
WorkingDirectory=/srv/app
```

makes the assumption explicit.

This does not make the application inherently better.

It makes the execution contract visible in one place.

---

## Failure example: mount missing

Cron:

```cron
0 2 * * * /usr/local/sbin/backup
```

Script must check:

```bash
mountpoint -q /mnt/backup
```

Systemd can add:

```ini
RequiresMountsFor=/mnt/backup
```

The script should still validate destination semantics.

Defense in depth is reasonable.

---

## Failure example: process hangs

Cron:

```text
future runs overlap unless lock exists
```

Add:

```bash
timeout
flock
```

Systemd:

```ini
TimeoutStartSec=30min
```

and unit state can prevent same-unit overlap.

Systemd provides more of the behavior declaratively.

---

## Failure example: root code compromise

Cron:

```cron
* * * * * root /srv/app/job
```

Systemd:

```ini
User=root
ExecStart=/srv/app/job
```

Both are insecure if `/srv/app/job` is writable by deployment users.

Scheduler technology does not repair trust-chain bugs.

---

## Failure example: log flood

Cron file output can fill disk.

Journald also consumes disk.

Both require retention policy.

There is no scheduler that removes the need for capacity management.

---

## Failure example: duplicate multi-host job

Cron on three hosts:

```text
three runs
```

Systemd timer on three hosts:

```text
three runs
```

Neither is cluster-aware by default.

Use distributed coordination.

---

## Failure example: missed run after outage

Traditional cron:

```text
missed
```

Cron + anacron:

```text
catch-up for suitable periodic jobs
```

Systemd timer with:

```ini
Persistent=true
```

can catch up.

This is a concrete feature difference.

---

## Failure example: server reboot at exact schedule time

If a daily task is operationally critical, do not rely solely on:

```text
server probably up at 02:00
```

Use:

```text
persistent timer
anacron
or
application reconciliation
```

depending on environment.

---

## Failure example: timer fires before dependency

Systemd ordering can reduce this.

But application readiness still matters.

A database service can be:

```text
active
```

while schema migration is incomplete.

The job should validate what it actually needs.

---

## `ConditionPathExists=`

Systemd can conditionally skip a service unless a path exists.

Example:

```ini
ConditionPathExists=/etc/company-report.conf
```

This can prevent meaningless execution when required configuration is absent.

But note the semantics:

```text
condition not met
```

is generally not equivalent to application failure.

If missing configuration should trigger an alert, explicit application validation may be better.

---

## `ConditionACPower=`

On systems and systemd versions supporting relevant conditions, power-related conditions can prevent heavy jobs from running on battery.

This can be useful for laptops.

But a skipped timer activation may affect missed-run expectations.

Always test behavior against the actual requirement.

Conditions are convenient, but they alter execution semantics.

---

## `ExecCondition=`

Systemd can run a condition command before `ExecStart`.

Example:

```ini
ExecCondition=/usr/bin/test -f /etc/company/enabled
```

This is more dynamic than static conditions.

Again, understand how condition exit status affects service result.

Use:

```bash
man systemd.service
```

for exact semantics.

---

## Pre- and post-commands

Systemd services can use:

```ini
ExecStartPre=
ExecStart=
ExecStartPost=
```

and stop-related commands.

This can express:

```text
preflight
main task
post-processing
```

But do not build a giant workflow engine from service directives.

Complex logic belongs in application code or a script.

---

## Why shell scripts still matter with systemd

Systemd can configure execution beautifully.

It is not a programming language.

If a job needs:

```text
loop over 100 items
conditional API logic
transaction handling
JSON parsing
retry strategy
complex validation
```

write a proper program or script.

Use systemd for:

```text
lifecycle
identity
resources
dependencies
scheduling
sandboxing
```

Use application code for business logic.

---

## Why shell scripts still matter with cron

The same principle applies.

Cron should remain:

```text
schedule -> program
```

not:

```text
schedule -> 400-character shell program
```

A boring scheduler definition is easier to audit.

---

## A clean cron production pattern

```cron
SHELL=/bin/sh
PATH=/usr/bin:/bin

15 3 * * * report /usr/local/libexec/company-report
```

Script:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

exec 9>/run/company-report.lock

if ! flock -n 9; then
    logger -t company-report \
        "event=skip reason=already_running"
    exit 0
fi

exec /opt/company-report/.venv/bin/python \
    /opt/company-report/report.py
```

This is a very strong design.

There is nothing inherently inferior about it.

---

## A clean systemd production pattern

Service:

```ini
[Unit]
Description=Company report
After=postgresql.service
Requires=postgresql.service

[Service]
Type=oneshot
User=report
Group=report
WorkingDirectory=/opt/company-report
ExecStart=/opt/company-report/.venv/bin/python /opt/company-report/report.py
TimeoutStartSec=20min
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/var/lib/company-report
```

Timer:

```ini
[Unit]
Description=Schedule company report

[Timer]
OnCalendar=*-*-* 03:15:00
Persistent=true
RandomizedDelaySec=5min

[Install]
WantedBy=timers.target
```

This is stronger when those additional controls are useful.

---

## Operational runbook for cron

A concise runbook:

```text
show schedule:
    crontab -l
    cat /etc/cron.d/JOB

test command:
    sudo -u USER /path/to/job

see scheduler log:
    journalctl -u cron

see application log:
    journalctl -t JOB
    or application log file

check overlap:
    pgrep -af JOB
    lslocks

check last success:
    stat /var/lib/JOB/last-success
```

Document this next to the application.

---

## Operational runbook for systemd timer

```text
show timer:
    systemctl status JOB.timer

next/last activation:
    systemctl list-timers JOB.timer

show definitions:
    systemctl cat JOB.timer
    systemctl cat JOB.service

test:
    systemctl start JOB.service

logs:
    journalctl -u JOB.service

result:
    systemctl show JOB.service -p Result -p ExecMainStatus

restart schedule:
    systemctl restart JOB.timer
```

The manager provides a standardized interface.

---

## Real-world choice matrix

A useful mental matrix:

| Requirement | Cron | Systemd Timer |
|---|---|---|
| Simple daily command | Excellent | Good |
| Portable across Unix | Excellent | Poor |
| Linux/systemd-only | Good | Excellent |
| Missed-run catch-up | Needs anacron/app logic | Native with `Persistent=` |
| Boot dependency ordering | Weak | Strong |
| Resource limits | External/manual | Strong |
| Sandboxing | External | Strong |
| Journald integration | Indirect | Native |
| Per-job unit state | Weak | Strong |
| One-line simplicity | Excellent | Weak |
| Relative-to-completion timing | Weak | Strong |
| Seconds-level interval | Poor | Strong |
| Team familiarity | Often strong | Environment-dependent |

This table is not a universal scorecard.

It is a way to identify which requirements should influence the decision.

---

## Prefer cron when simplicity is a feature

Use cron confidently when:

```text
one host
simple timing
simple script
low risk
good application logging
no boot dependency
no missed-run requirement
no special resource policy
```

Adding systemd does not automatically improve such a job.

Every extra abstraction has maintenance cost.

---

## Prefer systemd timer when execution policy is part of the requirement

Use systemd when the job needs:

```text
User=
WorkingDirectory=
EnvironmentFile=
RequiresMountsFor=
After=
TimeoutStartSec=
MemoryMax=
CPUWeight=
ProtectSystem=
PrivateTmp=
NoNewPrivileges=
Persistent=
RandomizedDelaySec=
```

These are not decorative features.

They encode real operational policy.

When many are needed, systemd becomes significantly cleaner than reproducing them through shell wrappers and external tools.

---

## Prefer application schedulers for business workflows

Neither cron nor systemd is ideal when the job needs:

```text
distributed locks
workflow dependencies
persistent retry
task history
manual retry UI
per-item status
multi-node execution
exactly-once business guarantees
```

Use:

```text
job queue
workflow engine
application scheduler
database-backed worker system
orchestrator
```

Cron/systemd can trigger or supervise that system.

---

## Prefer orchestrator scheduling inside Kubernetes

If workloads live in Kubernetes, use:

```text
CronJob
Job
concurrencyPolicy
history limits
resource requests/limits
service accounts
secrets
network policy
```

rather than host cron in most cases.

The scheduler closest to the workload lifecycle is usually easier to operate.

---

## Prefer cloud scheduler for cloud-native serverless tasks

For:

```text
invoke function every hour
publish message daily
trigger managed workflow
```

a cloud scheduler can be more appropriate than maintaining a Linux VM solely for cron.

Again, choose the scheduler aligned with the execution environment.

---

## Avoid scheduler stacking

A fragile design:

```text
cron
    -> systemd-run
        -> Docker
            -> application scheduler
                -> queue
```

Every layer can be valid individually.

Together they make troubleshooting difficult.

Use the fewest scheduling layers necessary.

A clean design might be:

```text
systemd timer
    -> service
        -> application
```

or:

```text
cron
    -> application scheduler
```

Know which layer owns timing.

---

## One scheduler should own one periodic trigger

For a logical task:

```text
daily backup
```

there should be a clear owner:

```text
cron
or
anacron
or
systemd timer
or
Kubernetes
or
cloud scheduler
```

not several "just in case".

Redundant schedulers usually create duplicates, not reliability.

High availability should be designed through coordination, not accidental duplicate timers.

---

## Scheduler migration checklist

Before migration:

```text
identify all current schedule sources
record exact timing
record timezone
record execution user
record working directory
record environment
record timeout behavior
record overlap behavior
record missed-run behavior
record logging
record alerting
record last-success monitoring
```

During migration:

```text
build target scheduler disabled
test underlying command
validate target schedule
test target execution context
verify logs
verify permissions
verify resource controls
verify missed-run behavior
```

Cutover:

```text
disable old schedule
enable new schedule
verify only one source remains
observe first production run
```

After:

```text
update runbook
update monitoring
remove temporary migration files
review next-run schedule
```

---

## Testing the exact future activation

Cron requires manual reasoning or external helpers.

Systemd:

```bash
systemd-analyze calendar 'Mon..Fri *-*-* 09:00:00'
```

can show the next event.

For complex calendar rules, this reduces configuration mistakes.

Always validate before enabling production timers.

---

## Testing timezone assumptions

Check:

```bash
timedatectl
```

Then:

```bash
systemd-analyze calendar 'daily'
```

Observe timezone in output.

For cron:

```bash
date
```

and inspect implementation-specific timezone configuration.

Never assume a server uses UTC because you intended it to.

Verify.

---

## Testing downtime catch-up

For production, do not casually shut down a server only to test a timer.

Use:

```text
disposable VM
staging host
container/VM snapshot test
```

or supported timestamp/state inspection.

The behavior you care about:

```text
timer due while inactive
machine returns
service starts
application remains idempotent
```

should be validated before relying on it for critical backups.

---

## Testing sandbox restrictions

A service with:

```ini
ProtectSystem=strict
ProtectHome=yes
```

may break because the application unexpectedly writes:

```text
/etc
/home
/usr
```

Test manually:

```bash
systemctl start company-report.service
```

Inspect:

```bash
journalctl -u company-report.service
```

Then grant only required paths:

```ini
ReadWritePaths=/var/lib/company-report
```

Do not disable the whole sandbox at the first permission error.

Use failures to discover actual write requirements.

---

## Iterative hardening

Start with a working service.

Then add one layer at a time:

```text
User=
NoNewPrivileges=
PrivateTmp=
ProtectHome=
ProtectSystem=
ReadWritePaths=
capability restrictions
resource limits
```

Test after each change.

This produces a smaller and better understood permission set.

---

## `systemd-analyze security` as a review aid

Run:

```bash
systemd-analyze security company-report.service
```

Review high-exposure items.

Ask:

```text
Does this service need root?
Does it need home access?
Does it need devices?
Does it need writable /usr?
Does it need network?
Does it need capabilities?
```

Do not chase a perfect score if it breaks legitimate functionality.

Security scores are heuristics.

---

## Cron can also be hardened externally

A cron job can run:

```text
inside a container
through bubblewrap
under firejail
with seccomp wrapper
under dedicated user
with capabilities
with cgroups
```

Cron is not incompatible with strong isolation.

It simply does not provide those controls in the cron entry itself.

This is an important distinction.

---

## Operational simplicity versus centralized policy

Cron philosophy:

```text
small scheduler
external tools do the rest
```

Systemd philosophy:

```text
service manager owns more lifecycle policy
```

Neither is universally superior.

Cron favors composability.

Systemd favors centralized declarative control.

The right choice depends on system architecture and team practice.

---

## A final production example

Requirement:

```text
daily encrypted backup
02:00 local time
catch up after downtime
backup volume must be mounted
run as dedicated backup user
maximum runtime 90 minutes
must not access home directories
log to journal
spread starts by up to 10 minutes across fleet
```

Systemd maps naturally.

Service:

```ini
[Unit]
Description=Encrypted application backup
RequiresMountsFor=/mnt/backup

[Service]
Type=oneshot
User=backup
Group=backup
ExecStart=/usr/local/libexec/encrypted-backup
TimeoutStartSec=90min
NoNewPrivileges=yes
ProtectHome=yes
PrivateTmp=yes
ProtectSystem=strict
ReadWritePaths=/mnt/backup /var/lib/company-backup
Nice=10
```

Timer:

```ini
[Unit]
Description=Schedule encrypted application backup

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
RandomizedDelaySec=10min

[Install]
WantedBy=timers.target
```

This is an example where systemd is clearly a strong fit.

---

## A final cron example

Requirement:

```text
check local certificate expiration once every morning
host always online
command takes one second
no special dependencies
result is already sent to monitoring
```

Cron:

```cron
15 8 * * * /usr/local/libexec/check-certificates
```

That is excellent.

Replacing it with two systemd files may not improve anything meaningful.

---

## Real operational maturity is scheduler-independent

A well-operated scheduled task has:

```text
documented purpose
clear owner
known execution user
version-controlled code
known dependencies
bounded runtime
correct concurrency policy
idempotent behavior where possible
observable failure
last-success monitoring
safe credentials
tested recovery
```

Cron can support that.

Systemd can support that.

Kubernetes can support that.

Cloud schedulers can support that.

The scheduler is only one layer.

---

## Questions to ask before choosing

Ask:

```text
Does exact wall-clock time matter?
Can the host be offline?
Should missed work catch up?
Can two runs overlap?
Does the job need dependencies?
Does it need a mount?
Does it need strict resource limits?
Does it need sandboxing?
Does it need per-run status?
Is the environment Linux/systemd-only?
Does portability matter?
Does the team already operate one scheduler well?
Is this actually a business workflow that belongs in a queue?
```

The answers usually make the choice obvious.

---

## Final perspective

Cron is small, mature, portable, and easy to understand.

Systemd timers are more expressive and integrate scheduling with the Linux service lifecycle.

Cron is strongest when the job is already self-contained and needs only:

```text
when to start
```

Systemd timers are strongest when the operating system must also define:

```text
who runs it
where it runs
what it depends on
how long it may run
what resources it may consume
what files it may access
how it is logged
what happens after downtime
```

The mistake is not choosing cron.

The mistake is asking cron to solve problems it does not model.

The mistake is not choosing systemd.

The mistake is using systemd complexity where a one-line cron entry would be clearer.

The practical rule is:

```text
simple periodic command
    -> cron is often enough

system workload with lifecycle requirements
    -> systemd timer is often stronger

missed coarse periodic maintenance
    -> anacron or persistent timer

distributed business workflow
    -> application scheduler or queue
```

A skilled Linux engineer should be comfortable with all of them.

The real goal is not to standardize every job on one scheduler.

The goal is to make scheduled work predictable, observable, secure, and correct.
