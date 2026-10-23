# Cron & Crontab — Deep Dive

<p align="center">
  <strong>Scheduling on Linux, from a five-field expression to production-grade automation.</strong>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Linux-System%20Administration-black?style=for-the-badge&logo=linux&logoColor=white" alt="Linux">
  <img src="https://img.shields.io/badge/Cron-Deep%20Dive-0f766e?style=for-the-badge" alt="Cron">
  <img src="https://img.shields.io/badge/Security-Privilege%20Aware-7f1d1d?style=for-the-badge" alt="Security">
  <img src="https://img.shields.io/badge/Level-Intermediate%20%E2%86%92%20Advanced-1d4ed8?style=for-the-badge" alt="Level">
</p>

---

Cron is one of the oldest pieces of Linux administration that still appears almost everywhere.

It looks simple:

```cron
*/5 * * * * /opt/jobs/task.sh
```

but production behavior depends on much more than the five time fields.

A scheduled command has an execution identity, a shell, a working directory, an environment, file descriptors, filesystem permissions, runtime dependencies, output channels, exit status, locking behavior, security boundaries, and failure modes.

This section studies all of those layers.

The goal is not to memorize crontab syntax.

The goal is to understand **what actually happens on Linux when a scheduled command is launched**, why cron jobs fail even when the same command works in a terminal, how privileged scheduled jobs become security risks, how to make recurring work reliable, and when cron should be replaced by anacron, systemd timers, or an application-level scheduler.

The material is deliberately example-heavy. Every major idea is connected to actual commands, system behavior, troubleshooting techniques, and production scenarios.

---

## Repository map

```text
Cron and Crontab/
├── README.md
├── 01-Cron-Architecture-and-Execution-Model.md
├── 02-Crontab-Syntax-and-Time-Matching.md
├── 03-User-Crontabs-and-System-Cron.md
├── 04-Cron-Environment-and-Shell-Behavior.md
├── 05-Cron-Directories-and-System-Jobs.md
├── 06-Output-Logging-and-Failure-Diagnosis.md
├── 07-Permissions-Security-and-Privilege-Escalation.md
├── 08-Reliable-Jobs-Locking-and-Concurrency.md
├── 09-Anacron-and-Missed-Job-Execution.md
└── 10-Cron-vs-Systemd-Timers-and-Real-World-Operations.md
```

The files are ordered intentionally.

They begin with execution architecture, move through scheduling syntax and system layout, then focus on environment, observability, security, concurrency, missed execution, and finally scheduler selection in modern Linux systems.

---

## What this section is designed to teach

After completing the material, you should be able to look at a line such as:

```cron
15 2 * * * root /usr/local/sbin/backup
```

and reason far beyond:

```text
runs every day at 02:15
```

You should be able to ask:

```text
Which cron implementation is running?
Where did this definition come from?
Which UID/GID will execute it?
Which shell parses the command?
What PATH will exist?
What is the current working directory?
What happens to stdout and stderr?
What happens if the script takes more than 24 hours?
What happens if the system is powered off at 02:15?
What if the script is writable by another user?
What if a parent directory is writable?
What if the command uses a relative executable?
What happens if the backup disk is not mounted?
How is failure detected?
How is success verified?
Could another server execute the same logical task?
Should this be cron at all?
```

That is the level of understanding this section targets.

---

# The execution model

A cron entry is not a command pasted into your interactive terminal.

A more realistic mental model is:

```text
                       +-------------------+
                       |    cron daemon    |
                       +---------+---------+
                                 |
                                 | time expression matches
                                 v
                       +-------------------+
                       | execution context |
                       | UID / GID / env   |
                       +---------+---------+
                                 |
                                 v
                       +-------------------+
                       |       shell       |
                       | usually /bin/sh   |
                       +---------+---------+
                                 |
                                 v
                       +-------------------+
                       | command / script  |
                       +---------+---------+
                                 |
                +----------------+----------------+
                |                                 |
                v                                 v
        +---------------+                 +---------------+
        | stdout/stderr |                 |  exit status  |
        +---------------+                 +---------------+
                |                                 |
                v                                 v
        logs / mail /                        monitoring /
        journald / file                     failure logic
```

The scheduler controls only part of the system.

The application, shell, operating system, filesystem, and external services determine the rest.

---

# Chapters

## `01-Cron-Architecture-and-Execution-Model.md`

The first chapter builds the execution model from the daemon outward.

It covers:

- what `cron` and `crond` actually do
- how cron evaluates jobs
- process creation
- user identity
- shell execution
- environment construction
- working directory behavior
- TTY absence
- stdout and stderr
- `@reboot`
- process trees
- missed execution
- timezone behavior
- basic security implications
- observing cron from Linux itself

Typical commands include:

```bash
systemctl status cron
ps -ef | grep '[c]ron'
pstree -ap
journalctl -u cron
```

The purpose of this chapter is to stop thinking of cron as a text file and start thinking of it as a **process execution system**.

---

## `02-Crontab-Syntax-and-Time-Matching.md`

This chapter goes deep into the part most people associate with cron: time expressions.

It covers:

```text
*
ranges
lists
steps
day of month
month
day of week
special expressions
```

and, more importantly, the edge cases behind them.

Examples:

```cron
*/5 * * * *
0 */4 * * *
15 8-18 * * 1-5
0 0 1,15 * *
```

The chapter explains why:

```cron
*/10 * * * *
```

means matching minute values:

```text
00 10 20 30 40 50
```

rather than:

```text
ten minutes after the previous process finished
```

It also examines the special relationship between:

```text
day-of-month
day-of-week
```

and why schedules involving both fields are often misunderstood.

Other topics include:

- special aliases such as `@daily`
- percent-sign behavior
- timezone effects
- DST
- schedule testing
- common syntax mistakes
- real-world examples

---

## `03-User-Crontabs-and-System-Cron.md`

Linux has several cron contexts.

A user's:

```bash
crontab -e
```

is not the same format as:

```text
/etc/crontab
```

or:

```text
/etc/cron.d/*
```

This chapter explains the difference in execution identity and syntax.

Personal crontab:

```cron
0 2 * * * /home/alice/bin/report
```

System crontab:

```cron
0 2 * * * alice /home/alice/bin/report
```

The extra field is not decorative.

It defines the user under which the command is executed.

Topics include:

- `crontab -e`
- `crontab -l`
- `crontab -r`
- `crontab -u`
- root crontab
- service accounts
- `/etc/crontab`
- `/etc/cron.d/`
- permissions and ownership
- least privilege
- migration between user and system cron
- duplicate definitions

---

## `04-Cron-Environment-and-Shell-Behavior.md`

This chapter addresses the classic problem:

> The command works manually but fails from cron.

The reason is usually not the schedule.

The execution environment is different.

Interactive shell:

```text
PATH
HOME
NVM
PYENV
virtualenv
SSH_AUTH_SOCK
locale
current directory
terminal
shell startup files
```

Cron:

```text
small controlled environment
no interactive shell
no terminal
different working directory
possibly /bin/sh
```

The chapter covers:

- `/bin/sh` versus Bash
- `SHELL=`
- `PATH`
- `HOME`
- environment variables
- quoting
- working directory
- Python virtual environments
- `nvm`
- `pyenv`
- `#!/usr/bin/env`
- locale
- timezone
- `MAILTO`
- redirection
- pipeline exit status
- `umask`
- PAM
- resource limits
- SSH agent behavior
- Docker
- database credentials
- containers
- reduced-environment reproduction

One of the most useful debugging techniques in the entire section is:

```bash
sudo -u alice env -i \
    HOME=/home/alice \
    USER=alice \
    LOGNAME=alice \
    PATH=/usr/bin:/bin \
    SHELL=/bin/sh \
    /opt/jobs/task.sh
```

If a job works in your shell but fails here, it probably has a hidden environment dependency.

---

## `05-Cron-Directories-and-System-Jobs.md`

Linux systems often contain:

```text
/etc/cron.d/
/etc/cron.hourly/
/etc/cron.daily/
/etc/cron.weekly/
/etc/cron.monthly/
```

These directories do not all use the same format.

A file under:

```text
/etc/cron.d/
```

contains scheduling syntax.

A file under:

```text
/etc/cron.daily/
```

is normally an executable program.

Example:

```text
/etc/cron.d/company
```

```cron
0 2 * * * root /usr/local/sbin/company-job
```

versus:

```text
/etc/cron.daily/company
```

```bash
#!/bin/sh
exec /usr/local/sbin/company-job
```

The chapter also studies `run-parts`.

One of the easiest mistakes to reproduce:

```text
/etc/cron.daily/backup.sh
```

may be skipped on systems where the default `run-parts` filename rules reject filenames containing dots.

Always verify:

```bash
run-parts --test /etc/cron.daily
```

The chapter also covers:

- package-owned jobs
- file ownership
- executable permissions
- symlinks
- remote filesystems
- mount validation
- duplicate systemd/cron scheduling
- package upgrades
- secure administrative layouts

---

## `06-Output-Logging-and-Failure-Diagnosis.md`

A missing backup does not automatically mean:

```text
cron failed
```

It may mean:

```text
cron never launched the job
cron launched it and the shell failed
the script started and exited
the script hung
the kernel killed it
a dependency failed
the process exited 0 but produced invalid data
```

This chapter builds a diagnostic workflow around evidence.

Important tools include:

```bash
journalctl
logger
strace
lsof
ss
ps
pstree
/proc
df
free
```

Examples:

```bash
journalctl -u cron --since "30 minutes ago"
```

```bash
strace -f -e execve \
    sudo -u appuser \
    /opt/app/bin/task
```

```bash
readlink /proc/PID/cwd
```

```bash
tr '\0' '\n' < /proc/PID/environ
```

The chapter also explains:

- stdout/stderr
- `2>&1`
- append versus overwrite
- log rotation
- cron mail
- exit codes
- pipeline failure
- `set -euo pipefail`
- buffering
- OOM kills
- signals
- SELinux
- AppArmor
- artifact validation
- false-success scenarios
- heartbeat monitoring
- last-success files
- structured logs
- metrics

A major theme is:

```text
process success != business success
```

A backup process can return `0` while writing to the wrong filesystem.

A `curl` command can return successfully while the server returned a business error.

A fresh file can contain invalid data.

Production monitoring must verify outcomes.

---

## `07-Permissions-Security-and-Privilege-Escalation.md`

Cron is an important Linux privilege boundary.

This entry:

```cron
* * * * * root /opt/scripts/backup.sh
```

becomes a privilege-escalation vulnerability if:

```text
/opt/scripts/backup.sh
```

is writable by an unprivileged user.

The chapter models security as a trust chain:

```text
cron definition
    ->
interpreter
    ->
script
    ->
configuration
    ->
helpers
    ->
runtime
    ->
filesystem
```

It covers:

- writable cron files
- writable scripts
- parent-directory permissions
- `namei -l`
- ACLs
- Linux capabilities
- PATH hijacking
- sourced configuration
- runtime import paths
- Python virtualenv trust
- Node.js dependency trust
- PHP application scheduling
- wildcard/option injection
- unsafe temporary files
- symlink attacks
- TOCTOU
- command injection
- secrets
- SSH credentials
- Docker socket privileges
- persistence detection
- SELinux/AppArmor
- least privilege
- service accounts
- audit methodology

Useful audit commands:

```bash
namei -l /usr/local/sbin/job
```

```bash
getfacl /usr/local/sbin/job
```

```bash
getcap /usr/local/bin/tool
```

```bash
sudo find /etc/cron.d \
    -type f \
    \( -perm -0020 -o -perm -0002 \) \
    -ls
```

The security rule underneath almost every example is:

> A lower-trust principal must not control executable behavior used by a higher-privileged scheduled process.

---

## `08-Reliable-Jobs-Locking-and-Concurrency.md`

Cron does not wait for the previous run.

If:

```cron
* * * * * /opt/jobs/sync
```

takes four minutes, multiple copies can exist.

The chapter starts with that simple observation and moves into real concurrency engineering.

The main local Linux tool is:

```bash
flock
```

Example:

```cron
* * * * * /usr/bin/flock -n /run/sync.lock /opt/jobs/sync
```

This means:

```text
if another instance owns the lock
skip this execution
```

The chapter explains:

- blocking locks
- non-blocking locks
- lock wait time
- `flock -w`
- descriptor locks
- `lslocks`
- stale lock files
- PID file problems
- `mkdir` locks
- race conditions
- atomic rename
- idempotency
- database constraints
- retries
- exponential backoff
- timeout
- deadlocks
- lock ordering
- lock scope
- shared/exclusive locks
- multi-host coordination
- database advisory locks
- distributed locks
- queues
- backpressure
- multi-server cron

One of the core lessons:

```text
locking != idempotency
```

A lock prevents simultaneous execution.

It does not guarantee that rerunning the same job later is safe.

Reliable systems usually need both.

---

## `09-Anacron-and-Missed-Job-Execution.md`

Traditional cron assumes the machine is running when the scheduled time arrives.

If:

```cron
0 3 * * * /opt/jobs/cleanup
```

and the laptop is powered off at 03:00, the run is missed.

Anacron solves a different scheduling problem:

```text
this job should run approximately once per day,
even if the machine was unavailable
```

The chapter covers:

- `/etc/anacrontab`
- periods
- delays
- job identifiers
- timestamp state
- `/var/spool/anacron`
- missed execution
- `cron.daily` integration
- boot-time scheduling
- `RANDOM_DELAY`
- `START_HOURS_RANGE`
- boot storms
- dependency readiness
- suspend/hibernate
- snapshots
- multi-host limitations
- catch-up semantics
- monitoring last successful execution

The most important conceptual distinction:

```text
cron:
    is the current time a match?

anacron:
    is this periodic job overdue?
```

Anacron is not a historical replay engine.

If the machine was offline for five daily periods, a due job is normally executed once rather than five historical jobs being replayed.

Applications that require historical backfill must track periods themselves.

---

## `10-Cron-vs-Systemd-Timers-and-Real-World-Operations.md`

The final chapter compares cron with modern systemd timers.

Cron:

```cron
0 2 * * * /opt/jobs/backup
```

Systemd:

```ini
[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
```

with:

```ini
[Service]
Type=oneshot
User=backup
ExecStart=/opt/jobs/backup
```

The comparison is practical rather than ideological.

Systemd provides features such as:

```text
Persistent=
RandomizedDelaySec=
OnBootSec=
OnUnitActiveSec=
OnUnitInactiveSec=
User=
WorkingDirectory=
EnvironmentFile=
TimeoutStartSec=
MemoryMax=
CPUWeight=
RequiresMountsFor=
ProtectSystem=
ProtectHome=
PrivateTmp=
NoNewPrivileges=
```

Cron remains excellent when the job is simple.

Systemd becomes stronger when lifecycle policy is part of the requirement.

The chapter also covers:

- `systemd-analyze calendar`
- `systemctl list-timers`
- journald
- service status
- one-shot service behavior
- dependency ordering
- cgroups
- sandboxing
- resource controls
- timer migration
- duplicate scheduler detection
- systemd user timers
- production runbooks
- scheduler selection

---

# Quick reference

## Personal crontab

Edit:

```bash
crontab -e
```

List:

```bash
crontab -l
```

Remove:

```bash
crontab -r
```

Another user:

```bash
sudo crontab -u alice -e
```

---

## Crontab field layout

```text
┌──────────── minute (0 - 59)
│ ┌────────── hour (0 - 23)
│ │ ┌──────── day of month (1 - 31)
│ │ │ ┌────── month (1 - 12)
│ │ │ │ ┌──── day of week
│ │ │ │ │
* * * * * command
```

Example:

```cron
30 2 * * * /opt/jobs/backup
```

```text
every day at 02:30
```

---

## Common expressions

```cron
* * * * *        every minute
*/5 * * * *      every matching five-minute mark
0 * * * *        every hour
0 0 * * *        daily at midnight
0 3 * * 0        Sunday at 03:00
0 9 * * 1-5      weekdays at 09:00
0 0 1 * *        first day of month at midnight
```

---

## Special schedules

Common implementations may support:

```text
@reboot
@hourly
@daily
@weekly
@monthly
@yearly
@annually
```

Always verify:

```bash
man 5 crontab
```

on the target system.

---

## Diagnose cron itself

```bash
systemctl status cron
```

or:

```bash
systemctl status crond
```

Logs:

```bash
journalctl -u cron
```

or:

```bash
journalctl -u crond
```

---

## Capture cron's environment

Temporary diagnostic entry:

```cron
* * * * * /usr/bin/env | /usr/bin/sort > /tmp/cron-env.txt
```

Compare:

```bash
env | sort > /tmp/interactive-env.txt
diff -u /tmp/cron-env.txt /tmp/interactive-env.txt
```

---

## Test the real user

```bash
sudo -u appuser /opt/jobs/task
```

Minimal environment:

```bash
sudo -u appuser env -i \
    HOME=/home/appuser \
    USER=appuser \
    LOGNAME=appuser \
    PATH=/usr/bin:/bin \
    SHELL=/bin/sh \
    /opt/jobs/task
```

---

## Inspect path permissions

```bash
namei -l /opt/jobs/task
```

Extended ACL:

```bash
getfacl /opt/jobs/task
```

Capabilities:

```bash
getcap /opt/jobs/task
```

---

## Logging

Combined output:

```cron
0 2 * * * /opt/jobs/backup >> /var/log/backup.log 2>&1
```

System log:

```bash
logger -t backup "backup started"
```

Read:

```bash
journalctl -t backup
```

---

## Locking

Non-blocking:

```bash
flock -n /run/job.lock /opt/jobs/job
```

Wait up to 30 seconds:

```bash
flock -w 30 /run/job.lock /opt/jobs/job
```

Inspect locks:

```bash
lslocks
```

---

## Periodic directories

Test:

```bash
run-parts --test /etc/cron.daily
```

Other directories:

```bash
run-parts --test /etc/cron.hourly
run-parts --test /etc/cron.weekly
run-parts --test /etc/cron.monthly
```

---

## Systemd timers

List:

```bash
systemctl list-timers --all
```

Test calendar:

```bash
systemd-analyze calendar 'Mon..Fri *-*-* 09:00:00'
```

Inspect:

```bash
systemctl status example.timer
systemctl status example.service
```

Logs:

```bash
journalctl -u example.service
```

---

# Troubleshooting map

When a scheduled task fails, do not immediately edit the crontab.

Classify the failure.

```text
Did scheduler attempt execution?
        |
        +-- NO
        |    |
        |    +-- daemon running?
        |    +-- correct crontab?
        |    +-- correct user?
        |    +-- valid syntax?
        |    +-- correct timezone?
        |    +-- cron.d filename/permissions?
        |    +-- run-parts selecting file?
        |
        +-- YES
             |
             +-- Did process start?
             |     |
             |     +-- executable exists?
             |     +-- executable permission?
             |     +-- parent traversal?
             |     +-- valid shebang?
             |     +-- interpreter exists?
             |
             +-- Process started
                   |
                   +-- PATH?
                   +-- HOME?
                   +-- working directory?
                   +-- runtime version?
                   +-- credentials?
                   +-- mount?
                   +-- DNS/network?
                   +-- database?
                   +-- permissions?
                   +-- SELinux/AppArmor?
                   +-- timeout?
                   +-- OOM?
                   +-- overlapping run?
```

Then ask the final question:

```text
Did the intended business result occur?
```

That last question is frequently forgotten.

---

# Production design principles

A production cron job should be boring.

The schedule should usually look like:

```cron
15 2 * * * /usr/local/libexec/company-backup
```

rather than:

```cron
15 2 * * * cd /srv/app && source ~/.bashrc && export X=... && if command; then another | pipeline; fi >> ...
```

Put execution logic in version-controlled scripts or applications.

Use explicit paths.

Control the environment.

Use least privilege.

Protect executable code.

Define output behavior.

Preserve exit status.

Use locks where overlap is unsafe.

Use idempotency where retries are possible.

Validate the artifact or business result.

Monitor the age of the last successful run.

Choose the scheduler according to workload semantics.

---

# Example production wrapper

A compact but solid pattern:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

JOB=company-sync
LOCK=/run/company-sync.lock
RUN_ID="$(date -u +%Y%m%dT%H%M%SZ)-$$"
START="$(date +%s)"

PATH=/usr/local/bin:/usr/bin:/bin
export PATH

exec 9>"$LOCK"

if ! flock -n 9; then
    logger -t "$JOB" \
        "run_id=$RUN_ID event=skip reason=already_running"
    exit 0
fi

finish() {
    rc=$?
    end=$(date +%s)

    if (( rc == 0 )); then
        priority=user.info
        event=finish
    else
        priority=user.err
        event=fail
    fi

    logger \
        -p "$priority" \
        -t "$JOB" \
        "run_id=$RUN_ID event=$event rc=$rc duration_seconds=$((end-START))"

    exit "$rc"
}

trap finish EXIT

logger -t "$JOB" \
    "run_id=$RUN_ID event=start"

/usr/bin/timeout \
    --kill-after=30s \
    20m \
    /opt/company/bin/sync
```

Cron:

```cron
*/15 * * * * /usr/local/libexec/run-company-sync
```

This design makes several properties explicit:

```text
controlled PATH
one active local run
observable skip
timeout
run identifier
duration
exit status
system logging
```

It is still small enough to understand quickly.

---

# Security model

The security model for scheduled execution can be summarized as:

```text
privilege of executing identity
        ×
control over executable behavior
        =
risk
```

A root job is not dangerous simply because it runs as root.

It becomes dangerous when lower-privileged users can influence:

```text
script
configuration
PATH
runtime
plugins
imports
working directory
temporary files
symlinks
wildcard-expanded filenames
helper commands
```

For privileged jobs, always inspect:

```bash
namei -l /path/to/job
```

not only:

```bash
ls -l /path/to/job
```

The entire directory path matters.

---

# Reliability model

Do not assume:

```text
scheduled once
=
executed once
```

A job can:

```text
be missed
run twice
overlap
be manually rerun
fail halfway
be killed
run on another server
restart after partial success
```

Reliable applications should survive those conditions.

That usually means some combination of:

```text
flock
idempotency
atomic rename
database constraints
transactions
checkpoints
timeouts
durable state
monitoring
distributed coordination
```

---

# Cron is not a business workflow engine

Cron is excellent at:

```text
periodic system work
maintenance
reconciliation
simple reports
cleanup
backups
health checks
```

Cron is weak as the primary engine for:

```text
distributed workflow
billing
exactly-once external effects
per-item retries
large job queues
workflow dependencies
multi-node task ownership
```

For those workloads, use:

```text
application scheduler
queue
workflow engine
Kubernetes Jobs
cloud scheduler
database-backed worker system
```

Cron can still trigger the higher-level system.

---

# Recommended study approach

Do not read these files as command cheat sheets.

Run the labs.

Create intentionally broken jobs.

Watch:

```bash
journalctl -fu cron
```

while they execute.

Compare environments.

Break `PATH`.

Use a wrong working directory.

Create a 90-second task scheduled every minute.

Watch processes overlap.

Add `flock`.

Kill the lock holder.

Inspect:

```bash
lslocks
```

Create a script in `/etc/cron.daily/` with a filename rejected by `run-parts`.

Use:

```bash
run-parts --test
```

to prove why it disappears.

The fastest way to understand cron is to observe the operating system while cron is doing real work.

---

# Suggested lab machine

A disposable Linux VM is strongly recommended.

Good options:

```text
Ubuntu Server
Debian
Rocky Linux
AlmaLinux
Fedora Server
```

The exact cron implementation differs between distributions.

That difference is useful.

Compare:

```text
service name
log location
cron package
run-parts behavior
anacron integration
systemd timers
```

Do not assume behavior learned on one distribution applies identically to all Linux systems.

---

# Commands worth knowing by the end

```bash
crontab
systemctl
journalctl
logger
run-parts
anacron
flock
lslocks
ps
pstree
pgrep
lsof
ss
strace
namei
stat
getfacl
getcap
find
timeout
systemd-analyze
```

Cron becomes much easier once it is connected to the rest of Linux rather than studied as an isolated scheduler.

---

# Final mental model

A production scheduled job is not:

```text
five time fields + command
```

It is:

```text
time
+
execution identity
+
environment
+
shell/interpreter
+
filesystem trust
+
working directory
+
dependencies
+
output
+
exit status
+
concurrency policy
+
retry semantics
+
idempotency
+
security boundary
+
monitoring
```

Cron provides only the first part directly.

Understanding the rest is what turns cron from a memorized command into real Linux administration.

---

<p align="center">
  <strong>Linux scheduling is easy to write.<br>
  Reliable Linux scheduling is a systems problem.</strong>
</p>
