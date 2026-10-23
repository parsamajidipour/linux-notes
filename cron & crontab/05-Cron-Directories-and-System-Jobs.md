# Cron Directories and System Jobs

A Linux system can contain several different sources of scheduled work at the same time.

A user may have a personal crontab. The operating system may have `/etc/crontab`. Packages may install files under `/etc/cron.d/`. Distribution-maintained scripts may live in `/etc/cron.daily/`, `/etc/cron.hourly/`, `/etc/cron.weekly/`, or `/etc/cron.monthly/`. On some systems, `anacron` participates in running the periodic directories. On others, systemd timers have replaced part of that responsibility.

All of these mechanisms are related, but they are not interchangeable.

Understanding the difference matters for reliability and security. A command that is valid in a user crontab may be invalid in `/etc/cron.d/`. A shell script that runs perfectly when executed directly may never be selected by `run-parts`. A package-installed daily job may not run at the exact time shown in `/etc/crontab` because `anacron` controls the actual execution. A file with the correct command may be ignored because its filename, owner, or mode violates the scheduler's rules.

This chapter examines the directory-level architecture behind system cron jobs and shows how Linux distributions use it in practice.

---

## The system-wide cron layout

A typical Debian or Ubuntu system may contain:

```text
/etc/crontab
/etc/cron.d/
/etc/cron.hourly/
/etc/cron.daily/
/etc/cron.weekly/
/etc/cron.monthly/
```

A quick inspection:

```bash
ls -ld \
    /etc/crontab \
    /etc/cron.d \
    /etc/cron.hourly \
    /etc/cron.daily \
    /etc/cron.weekly \
    /etc/cron.monthly
```

may produce output similar to:

```text
-rw-r--r-- 1 root root 1136 Sep 10 08:21 /etc/crontab
drwxr-xr-x 2 root root 4096 Sep 10 08:21 /etc/cron.d
drwxr-xr-x 2 root root 4096 Sep 10 08:21 /etc/cron.hourly
drwxr-xr-x 2 root root 4096 Sep 10 08:21 /etc/cron.daily
drwxr-xr-x 2 root root 4096 Sep 10 08:21 /etc/cron.weekly
drwxr-xr-x 2 root root 4096 Sep 10 08:21 /etc/cron.monthly
```

The exact contents depend on distribution, installed packages, and administrative changes.

On another system:

```bash
find /etc -maxdepth 2 \
    \( -name 'crontab' -o -name 'cron.*' \) \
    -print
```

might reveal additional files such as:

```text
/etc/anacrontab
/etc/cron.deny
/etc/cron.allow
```

or distribution-specific directories.

The important architectural idea is that there are two broad models.

One model stores complete schedule definitions:

```text
/etc/crontab
/etc/cron.d/*
user crontabs
```

The other model stores executable programs grouped by frequency:

```text
/etc/cron.hourly/*
/etc/cron.daily/*
/etc/cron.weekly/*
/etc/cron.monthly/*
```

The second model usually depends on another scheduler entry that calls a program such as `run-parts` to execute the contents of the directory.

That means `/etc/cron.daily/` is not itself a scheduler.

It is a directory of jobs.

Something else decides when that directory is processed.

---

## `/etc/crontab` is a system crontab

Inspect it:

```bash
cat /etc/crontab
```

A Debian-family system may contain something structurally similar to:

```cron
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
```

Do not assume those exact times exist on every machine. Distribution versions differ, and some installations use systemd timers instead.

The important syntax difference from a personal crontab is the username field:

```text
minute hour day-of-month month day-of-week user command
```

For example:

```cron
30 2 * * * root /usr/local/sbin/nightly-backup
```

means:

```text
02:30 every day
run as user root
execute /usr/local/sbin/nightly-backup
```

A personal user crontab does not include the username field.

This is valid in `/etc/crontab`:

```cron
*/10 * * * * backup /opt/backup/run
```

but the same text is wrong in a user crontab because `backup` would be interpreted as part of the command.

Conversely, this is appropriate in a user's crontab:

```cron
*/10 * * * * /opt/backup/run
```

but incomplete in a system crontab format that requires an explicit user.

That difference is simple but operationally important.

---

## System crontabs separate scheduling from execution identity

One advantage of `/etc/crontab` and `/etc/cron.d/` is that they can schedule commands under different accounts.

For example:

```cron
15 1 * * * postgres /usr/local/libexec/db-maintenance
30 1 * * * backup   /usr/local/libexec/archive-data
45 1 * * * root     /usr/local/sbin/prune-snapshots
```

This is useful because system tasks do not all need root.

A PostgreSQL maintenance script may need only the database service account:

```bash
sudo -u postgres /usr/local/libexec/db-maintenance
```

A backup job might need read access to specific application data and write access to a backup destination, but no ability to modify the rest of the operating system.

The scheduled account should therefore be chosen according to least privilege.

A weak design is:

```cron
* * * * * root /opt/app/run-report
```

simply because "cron jobs are system jobs".

A stronger design asks:

```text
What resources does this task actually need?
Which UID should own those permissions?
Does the task require root at all?
```

Cron supports privilege separation naturally. It should be used.

---

## `/etc/cron.d/` extends the system crontab model

The directory:

```text
/etc/cron.d/
```

contains files whose entries usually follow system-crontab syntax.

Example:

```bash
sudo cat /etc/cron.d/example-maintenance
```

could contain:

```cron
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

5 * * * * appuser /usr/local/libexec/example/check-state
40 3 * * * root /usr/local/libexec/example/cleanup
```

The presence of a username column is essential.

A common mistake is to copy an entry directly from:

```bash
crontab -l
```

into `/etc/cron.d/myjob`.

Suppose the original user crontab contains:

```cron
0 3 * * * /home/alice/bin/report
```

If the administrator copies it verbatim into `/etc/cron.d/report`, a system cron parser may interpret:

```text
/home/alice/bin/report
```

as the username field.

The entry is then invalid or ignored.

The correct form is:

```cron
0 3 * * * alice /home/alice/bin/report
```

The inverse mistake also occurs: copying an `/etc/cron.d/` line into `crontab -e` and leaving the username in place.

Always identify the crontab format before editing it.

---

## Why `/etc/cron.d/` exists

A single `/etc/crontab` becomes difficult to manage if every installed package modifies it.

Suppose five packages all need scheduled tasks.

Directly modifying one central file creates several problems:

```text
package installation must edit shared state
package removal must locate and remove its own lines
upgrades can cause merge conflicts
administrators and packages can overwrite each other
ownership becomes unclear
```

`/etc/cron.d/` solves this by allowing each package or administrator to own a separate file.

For example:

```text
/etc/cron.d/php
/etc/cron.d/cert-maintenance
/etc/cron.d/app-backup
/etc/cron.d/database-cleanup
```

Each file can be installed, upgraded, disabled, or removed independently.

This mirrors a broader Linux configuration pattern seen in directories such as:

```text
/etc/sysctl.d/
/etc/logrotate.d/
/etc/sudoers.d/
/etc/systemd/system/*.d/
```

A drop-in directory avoids turning one global file into a conflict-prone configuration database.

---

## Filenames under `/etc/cron.d/` matter

Cron implementations often apply restrictions to files under `/etc/cron.d/`.

Temporary editor files, backup files, hidden files, or filenames containing unsupported characters may be ignored.

Examples that may be problematic:

```text
/etc/cron.d/backup~
/etc/cron.d/.backup.swp
/etc/cron.d/app.disabled
/etc/cron.d/#app#
```

A safer filename is simple:

```text
/etc/cron.d/app-backup
```

The exact accepted pattern is implementation-dependent.

On Debian-family systems, `run-parts` also has its own filename-selection rules, which are separate from cron's handling of `/etc/cron.d/`.

Do not infer acceptance from:

```bash
ls /etc/cron.d/
```

A visible file is not necessarily a parsed file.

When a job is mysteriously ignored, inspect:

```bash
man 5 crontab
man 8 cron
```

for the installed implementation, and verify logs.

A package can also expose cron's parsing decision through messages in the system journal or syslog.

---

## Ownership and permissions of `/etc/cron.d/` files are security-sensitive

A system cron definition can cause commands to run as root.

That makes the file itself part of the privilege boundary.

Inspect:

```bash
ls -l /etc/cron.d/
```

A normal system-owned file may look like:

```text
-rw-r--r-- 1 root root 220 Sep 10 09:15 app-backup
```

A dangerous file might be:

```text
-rw-rw-r-- 1 root developers 220 Sep 10 09:15 app-backup
```

If members of `developers` can edit a file containing:

```cron
* * * * * root /usr/local/sbin/backup
```

they can change the command to anything root can execute.

That is direct privilege escalation.

The risk is not limited to the crontab file itself.

Suppose:

```cron
* * * * * root /opt/company/jobs/backup.sh
```

Check the full path:

```bash
namei -l /opt/company/jobs/backup.sh
```

If `/opt/company/jobs/` is writable by an unprivileged group, the root cron definition remains unsafe.

Security review must follow the chain:

```text
cron definition
  -> shell/interpreter
  -> script
  -> sourced files
  -> configuration
  -> executable dependencies
  -> writable parent directories
```

A root-owned crontab pointing to an attacker-writable script is still attacker-controlled execution.

---

## Cron may reject insecure files

Some cron implementations reject or ignore files with unexpected ownership or permissions.

The exact rules differ.

Instead of assuming, inspect:

```bash
stat /etc/cron.d/app-backup
```

and compare with known working files:

```bash
stat /etc/cron.d/* 2>/dev/null
```

Typical administrative practice is:

```bash
sudo chown root:root /etc/cron.d/app-backup
sudo chmod 0644 /etc/cron.d/app-backup
```

The scheduled executable itself may use stricter permissions:

```bash
sudo chown root:root /usr/local/sbin/app-backup
sudo chmod 0750 /usr/local/sbin/app-backup
```

if only privileged users should run it.

The key distinction is:

```text
readability of the cron definition
executability of the target command
writability of trusted files
```

These are separate permission questions.

---

## A package-style cron file

Suppose a server package named `metrics-collector` needs to run a maintenance command every fifteen minutes.

A clean `/etc/cron.d/metrics-collector` might be:

```cron
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

*/15 * * * * metrics /usr/lib/metrics-collector/rotate-state
```

The executable:

```text
/usr/lib/metrics-collector/rotate-state
```

could be owned by root:

```bash
ls -l /usr/lib/metrics-collector/rotate-state
```

```text
-rwxr-xr-x 1 root root 1840 Sep 10 11:01 /usr/lib/metrics-collector/rotate-state
```

while the runtime data is owned by `metrics`:

```text
/var/lib/metrics-collector/
/var/log/metrics-collector/
```

This gives a useful separation:

```text
root owns code and schedule definition
metrics user owns runtime data
cron launches job as metrics
```

A compromise of the application account does not automatically permit changing the root-owned scheduled command.

That is stronger than placing both script and cron file inside a writable application directory.

---

## Periodic directories are executable job collections

Consider:

```text
/etc/cron.daily/
```

A file there does not normally contain a five-field cron schedule.

Instead, it is an executable program or script.

Example:

```bash
sudo tee /etc/cron.daily/example-cleanup >/dev/null <<'EOF'
#!/bin/sh
set -eu

/usr/bin/find /var/cache/example \
    -type f \
    -mtime +14 \
    -delete
EOF

sudo chmod 0755 /etc/cron.daily/example-cleanup
sudo chown root:root /etc/cron.daily/example-cleanup
```

There is no line such as:

```cron
0 4 * * * ...
```

inside the file.

The frequency comes from the directory:

```text
cron.daily
```

and from whichever scheduler invokes that directory.

The script itself describes only the work.

That distinction is important when migrating jobs between direct crontab entries and periodic directories.

---

## `run-parts` is the mechanism behind many periodic directories

On Debian-family systems, periodic directories are commonly executed by:

```bash
run-parts
```

Inspect it:

```bash
command -v run-parts
```

Possible output:

```text
/usr/bin/run-parts
```

Its job is conceptually simple:

```text
inspect a directory
select eligible filenames
execute each selected file
```

But filename selection and ordering rules matter.

You can test a directory without executing anything:

```bash
run-parts --test /etc/cron.daily
```

Example output:

```text
/etc/cron.daily/apt-compat
/etc/cron.daily/dpkg
/etc/cron.daily/logrotate
/etc/cron.daily/man-db
```

This is one of the most useful commands when debugging periodic jobs.

A file can exist in the directory and still not appear in `--test`.

That immediately tells you the scheduler may never attempt to execute it.

---

## Filename rules can make a valid script invisible

Create a laboratory directory:

```bash
mkdir -p /tmp/run-parts-lab
```

Add:

```bash
cat > /tmp/run-parts-lab/backup.sh <<'EOF'
#!/bin/sh
echo backup
EOF

chmod +x /tmp/run-parts-lab/backup.sh
```

Now run:

```bash
run-parts --test /tmp/run-parts-lab
```

On common Debian `run-parts` defaults, `backup.sh` may not appear because the dot in the filename does not match the traditional allowed filename policy.

Rename it:

```bash
mv /tmp/run-parts-lab/backup.sh /tmp/run-parts-lab/backup
```

Test again:

```bash
run-parts --test /tmp/run-parts-lab
```

Now:

```text
/tmp/run-parts-lab/backup
```

may appear.

This surprises administrators because shell scripts are often conventionally named with `.sh`.

Under a `run-parts` directory, a simple extension can determine whether the job executes at all.

Always test:

```bash
run-parts --test DIRECTORY
```

instead of assuming that every executable file is selected.

---

## `run-parts --list` and `--test`

Depending on implementation, useful modes can include:

```bash
run-parts --test /etc/cron.daily
run-parts --list /etc/cron.daily
```

The exact available switches should be checked with:

```bash
run-parts --help
man run-parts
```

`--test` is especially useful because it shows what would be executed without actually running the jobs.

That makes it safe for production inspection.

For example, after installing:

```text
/etc/cron.daily/company-backup
```

verify:

```bash
run-parts --test /etc/cron.daily | grep company-backup
```

If there is no output, investigate:

```text
filename
execute permission
implementation-specific selection rules
```

before waiting until the next morning.

---

## Executable permission matters

Create:

```bash
sudo install -o root -g root -m 0644 \
    /dev/null /etc/cron.daily/not-executable
```

Even if the file contains a valid shebang and commands, a runner expecting executable files may skip it or fail to execute it.

Inspect:

```bash
ls -l /etc/cron.daily/not-executable
```

```text
-rw-r--r-- 1 root root ...
```

Set:

```bash
sudo chmod 0755 /etc/cron.daily/not-executable
```

Then test the directory again:

```bash
run-parts --test /etc/cron.daily
```

Permissions are part of job discovery.

A common diagnostic sequence is:

```bash
ls -l /etc/cron.daily
run-parts --test /etc/cron.daily
file /etc/cron.daily/myjob
head -n 1 /etc/cron.daily/myjob
```

These checks answer four different questions:

```text
Does the file exist?
Is it executable?
Will run-parts select it?
Does it have a valid interpreter?
```

---

## The shebang is required for direct script execution

Suppose:

```text
/etc/cron.daily/report
```

contains:

```bash
echo "running report"
python3 /opt/report/report.py
```

and is executable.

If the scheduler executes it directly with:

```bash
execve("/etc/cron.daily/report", ...)
```

the kernel needs to know how to interpret the file.

A shell script should begin with a shebang:

```bash
#!/bin/sh
```

or:

```bash
#!/usr/bin/env bash
```

depending on the script's language requirements.

A complete file:

```bash
#!/bin/sh
set -eu

/usr/bin/python3 /opt/report/report.py
```

Without a valid shebang, direct execution may fail with an error such as:

```text
Exec format error
```

Testing a script as:

```bash
sh /etc/cron.daily/report
```

can hide this problem because you explicitly provide an interpreter.

Test it the way the scheduler will:

```bash
sudo /etc/cron.daily/report
```

That is a more meaningful validation.

---

## The periodic directory does not imply an exact execution time

The name:

```text
/etc/cron.daily/
```

means approximately:

```text
these jobs belong to the daily execution class
```

It does not universally mean:

```text
every job runs exactly at 06:25
```

A traditional `/etc/crontab` may invoke:

```bash
run-parts /etc/cron.daily
```

at a fixed time.

But if `anacron` is installed, `/etc/crontab` may deliberately avoid running daily jobs directly:

```cron
25 6 * * * root test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
```

The condition means:

```text
if anacron is not executable,
then run the daily directory here
```

If anacron exists, another mechanism handles those periodic jobs.

That matters on laptops and intermittently running machines.

Traditional cron misses a 06:25 job if the computer is powered off at 06:25.

Anacron is designed to execute jobs that were missed because the machine was unavailable.

The details of anacron belong in a dedicated chapter, but this dependency must be recognized when investigating `/etc/cron.daily/`.

---

## Trace who actually invokes `/etc/cron.daily`

Instead of guessing, search configuration:

```bash
grep -R "/etc/cron.daily" \
    /etc/crontab \
    /etc/cron.d \
    /etc/anacrontab \
    2>/dev/null
```

Also inspect timers:

```bash
systemctl list-timers --all
```

On a modern Linux system, periodic package maintenance may be driven partly or entirely by systemd timers.

For example, a package may have:

```text
apt-daily.timer
apt-daily-upgrade.timer
logrotate.timer
```

rather than relying on a classic daily cron script.

The existence of `/etc/cron.daily/` does not prove every periodic system task uses it.

Linux distributions evolve. The correct source of truth is the target machine's configuration.

---

## `/etc/cron.hourly/`

Hourly jobs are appropriate for low-cost maintenance tasks that genuinely need hourly execution.

Example:

```bash
sudo tee /etc/cron.hourly/app-cache-clean >/dev/null <<'EOF'
#!/bin/sh
set -eu

/usr/bin/find /var/cache/myapp \
    -type f \
    -mmin +180 \
    -delete
EOF

sudo chown root:root /etc/cron.hourly/app-cache-clean
sudo chmod 0755 /etc/cron.hourly/app-cache-clean
```

Verify discovery:

```bash
run-parts --test /etc/cron.hourly | grep app-cache-clean
```

Run manually:

```bash
sudo /etc/cron.hourly/app-cache-clean
```

The script should be idempotent if possible.

That means running it twice should not corrupt state.

For a cleanup command:

```bash
find ... -delete
```

the second run usually finds nothing and exits normally.

Idempotence is especially valuable in periodic directories because administrators may run them manually during diagnostics.

---

## `/etc/cron.daily/`

Daily jobs are commonly used for tasks such as:

```text
log maintenance
package housekeeping
database statistics
cache cleanup
report generation
temporary-file pruning
backup metadata
index updates
```

A good daily task should tolerate execution at a slightly variable time.

For example:

```bash
#!/bin/sh
set -eu

TODAY=$(/usr/bin/date -u +%F)
OUTPUT="/var/lib/reports/daily-${TODAY}.json"

if [ -e "$OUTPUT" ]; then
    exit 0
fi

/usr/local/bin/build-daily-report > "$OUTPUT.tmp"
/bin/mv "$OUTPUT.tmp" "$OUTPUT"
```

This job avoids blindly appending duplicate output if it is executed twice.

It also writes to a temporary file first, then renames it into place after successful completion.

That helps avoid a partially written final artifact.

Periodic jobs should be designed around scheduler realities, not only around the happy path.

---

## `/etc/cron.weekly/`

Weekly tasks often include:

```text
deep cleanup
large indexes
archive rotation
full integrity checks
slow reports
longer maintenance windows
```

A job that scans millions of files may be too expensive for daily execution.

Example:

```bash
#!/bin/sh
set -eu

/usr/bin/nice -n 10 \
    /usr/bin/find /srv/archive \
    -type f \
    -mtime +365 \
    -print \
    > /var/lib/archive/old-files.txt
```

The use of `nice` can reduce CPU scheduling priority, but it does not limit I/O by itself.

On a storage-heavy server, a weekly job can still impact production workloads.

Cron frequency design should therefore consider:

```text
CPU cost
I/O cost
database locks
network bandwidth
duration
concurrency
business traffic
```

"Weekly" is not automatically safe.

---

## `/etc/cron.monthly/`

Monthly directories are suitable for rare maintenance where exact calendar semantics are not highly specific.

Examples:

```text
long-term archive
certificate inventory reports
large audit reports
slow cleanup
billing support exports
```

If a business process requires:

```text
the last business day of every month
at 17:00 local time
```

a monthly directory is too vague.

That kind of requirement belongs in an explicit scheduler expression or a more capable scheduling system.

Periodic directories intentionally trade scheduling precision for administrative simplicity.

---

## Ordering inside periodic directories

`run-parts` generally executes selected files in a deterministic lexical order, but exact sorting behavior should be verified for the installed implementation.

Administrators sometimes try to force order with names such as:

```text
10-prepare
20-process
30-cleanup
```

This may work, but it introduces coupling between otherwise independent jobs.

If `20-process` requires `10-prepare` to succeed, they are not really separate scheduled jobs.

A more reliable design is a single orchestration script:

```bash
#!/bin/sh
set -eu

/usr/local/libexec/report/prepare
/usr/local/libexec/report/process
/usr/local/libexec/report/cleanup
```

with one scheduled entry.

This provides explicit error propagation and makes dependency order obvious.

Directory ordering is useful for independent jobs, but it should not be abused as a workflow engine.

---

## One failing `run-parts` job does not necessarily stop the directory

The behavior depends on `run-parts` options and implementation.

Suppose:

```text
/etc/cron.daily/job-a
/etc/cron.daily/job-b
/etc/cron.daily/job-c
```

and `job-b` exits with status `1`.

Administrators sometimes assume:

```text
job-c will not run
```

or the opposite:

```text
failure is ignored completely
```

Do not guess.

Build a lab.

```bash
mkdir -p /tmp/run-parts-status
```

Create:

```bash
cat > /tmp/run-parts-status/10-ok <<'EOF'
#!/bin/sh
echo ok-1
exit 0
EOF

cat > /tmp/run-parts-status/20-fail <<'EOF'
#!/bin/sh
echo fail
exit 7
EOF

cat > /tmp/run-parts-status/30-ok <<'EOF'
#!/bin/sh
echo ok-2
exit 0
EOF

chmod +x /tmp/run-parts-status/*
```

Run:

```bash
run-parts /tmp/run-parts-status
echo "run-parts rc=$?"
```

Observe the actual behavior on the system.

Then check:

```bash
man run-parts
```

This is better than building operational assumptions from memory.

---

## `run-parts --report` changes output behavior

Some system cron configurations use:

```bash
run-parts --report /etc/cron.daily
```

rather than:

```bash
run-parts /etc/cron.daily
```

On Debian's implementation, `--report` reports script names when they produce output, which helps cron mail or logs identify the source.

The exact behavior should be verified with:

```bash
man run-parts
```

For administrators, this is another reminder that the caller matters.

The jobs in `/etc/cron.daily/` may be identical, but:

```text
how they are executed
what options are passed
how output is captured
```

depends on the parent scheduler configuration.

---

## Package managers commonly own periodic jobs

List package ownership on Debian-family systems:

```bash
dpkg -S /etc/cron.daily/logrotate 2>/dev/null
```

or on RPM-family systems:

```bash
rpm -qf /etc/cron.daily/somejob
```

This helps answer:

```text
Was this file created by an administrator?
Or does an installed package own it?
```

That distinction matters before editing.

If a package owns the file, modifying it directly may be overwritten by an upgrade or may create a configuration-file conflict.

A better customization strategy may be:

```text
package-specific configuration
a separate local cron file
a systemd override
disabling the packaged job through supported configuration
```

Do not treat every file under `/etc/cron.*` as locally authored.

---

## Package upgrades can change scheduling architecture

A package may historically install:

```text
/etc/cron.daily/foo
```

and later migrate to:

```text
foo.timer
foo.service
```

After an upgrade, both mechanisms could temporarily coexist if migration or local modifications are unusual.

That can create duplicate execution.

Investigate with:

```bash
grep -R "foo" /etc/cron* 2>/dev/null
systemctl list-timers --all | grep -i foo
systemctl list-unit-files | grep -i foo
```

Then inspect package files:

```bash
dpkg -L foo
```

or:

```bash
rpm -ql foo
```

Duplicate scheduling is a real operational failure mode.

It can cause:

```text
double backups
duplicate emails
two cleanup processes racing
database maintenance overlap
repeated API calls
corrupted lock state
```

Whenever migrating a scheduled task, search all scheduler layers.

---

## Disabling a periodic job safely

One tempting method is:

```bash
chmod -x /etc/cron.daily/somejob
```

This may work with runners that execute only executable files.

Another is renaming:

```bash
mv /etc/cron.daily/somejob /etc/cron.daily/somejob.disabled
```

If `run-parts` rejects filenames containing a dot, that can prevent execution.

But ad-hoc renaming of package-owned files can confuse package management.

Prefer the application's supported disable mechanism when available.

Before changing anything, identify ownership:

```bash
dpkg -S /etc/cron.daily/somejob
```

or:

```bash
rpm -qf /etc/cron.daily/somejob
```

Then inspect the package documentation.

For a local job, an explicit administrative move to a dedicated disabled directory can be clearer:

```bash
sudo mkdir -p /etc/cron.disabled
sudo mv /etc/cron.daily/company-backup /etc/cron.disabled/
```

Now there is no ambiguity about whether the filename happens to match `run-parts` rules.

---

## Never place secrets directly in system cron definitions

This is a bad design:

```cron
0 2 * * * root DB_PASSWORD='secret' /opt/app/backup
```

Potential exposure includes:

```text
configuration backups
support bundles
root-readable but widely copied files
screen sharing
configuration management output
accidental Git commits
debugging logs
```

The cron definition should generally point to a trusted script:

```cron
0 2 * * * root /usr/local/sbin/app-backup
```

The script can obtain secrets through:

```text
root-readable config file
service-account credential file
secret manager
kernel keyring
application-specific credential store
```

with permissions appropriate to the deployment.

The filesystem layout should separate public scheduling metadata from sensitive runtime credentials.

---

## Avoid writable application trees for privileged cron scripts

Developers often deploy code under:

```text
/var/www/app
/opt/app
/srv/app
```

and make it writable by a deployment account.

Then someone creates:

```cron
* * * * * root /var/www/app/scripts/maintenance.sh
```

This makes the deployment account a root code-authority.

If that is not intentional, move the privileged wrapper to a root-controlled location:

```text
/usr/local/sbin/app-maintenance
```

and limit what it delegates.

For example:

```bash
#!/bin/sh
set -eu

exec /usr/bin/sudo -u appuser \
    /var/www/app/bin/maintenance \
    --mode safe
```

Even this wrapper must be reviewed carefully, because arguments, configuration files, imported code, and writable paths can reintroduce privilege escalation.

A stronger architectural rule is:

```text
root executes only root-controlled code
application users execute application-controlled code
```

Crossing that boundary should be explicit.

---

## Symlinks require care

Suppose:

```text
/etc/cron.daily/company-backup
```

is a symlink:

```bash
ls -l /etc/cron.daily/company-backup
```

```text
company-backup -> /opt/company/current/bin/backup
```

Now the actual code can change when:

```text
/opt/company/current
```

is repointed during deployment.

That may be desirable.

It may also allow a deployment user to change what root executes.

Inspect:

```bash
readlink -f /etc/cron.daily/company-backup
namei -l "$(readlink -f /etc/cron.daily/company-backup)"
```

When symlinks cross privilege boundaries, review who controls:

```text
the symlink
every parent directory
the final target
deployment pointer directories
```

Convenience does not remove trust requirements.

---

## Avoid NFS and remote filesystem assumptions for scheduler definitions

Placing executable cron scripts on network filesystems can create failure modes such as:

```text
mount not available at execution time
network timeout
root_squash behavior
changed file ownership semantics
stale file handles
remote compromise affecting local privileged execution
```

For critical root jobs, keeping the scheduling wrapper on a local root-controlled filesystem is usually easier to trust.

The wrapper can then access remote data intentionally.

Example:

```bash
#!/bin/sh
set -eu

MOUNT=/mnt/archive

/usr/bin/mountpoint -q "$MOUNT" || {
    /usr/bin/logger -p user.err -t archive-job \
        "$MOUNT is not mounted"
    exit 1
}

/usr/bin/rsync -a /srv/data/ "$MOUNT/data/"
```

This avoids silently writing into an unmounted local directory that merely happens to exist at the mountpoint path.

---

## A missing mount can create a dangerous false success

Suppose:

```cron
0 2 * * * root /usr/bin/rsync -a /srv/data/ /backup/
```

and `/backup` is normally a mounted disk.

If the disk is not mounted, `/backup` may still exist as an ordinary directory on the root filesystem.

`rsync` succeeds.

The backup appears successful.

But the data was written to the wrong filesystem.

A robust system job verifies infrastructure assumptions:

```bash
#!/bin/sh
set -eu

DEST=/backup

if ! /usr/bin/mountpoint -q "$DEST"; then
    /usr/bin/logger -p user.err -t backup \
        "destination is not a mountpoint: $DEST"
    exit 1
fi

exec /usr/bin/rsync -a /srv/data/ "$DEST/data/"
```

This is an example of why system cron design must include operating-system state, not just scheduling syntax.

---

## Jobs in periodic directories should control their own working directory

A periodic script should not assume the scheduler invokes it from the script's directory.

This is fragile:

```bash
#!/bin/sh
./cleanup-cache
```

because `./cleanup-cache` refers to the current working directory.

Use:

```bash
#!/bin/sh
exec /usr/local/libexec/app/cleanup-cache
```

or establish a directory:

```bash
#!/bin/sh
set -eu

cd /srv/app
exec ./bin/cleanup-cache
```

A traditional `/etc/crontab` entry may explicitly execute:

```bash
cd / && run-parts /etc/cron.daily
```

which means every script begins with `/` as its current directory unless it changes it itself.

This is deliberate.

It discourages accidental dependence on arbitrary caller state.

---

## A job in `/etc/cron.daily/` can still use locks

Directory-level grouping does not prevent overlap.

Imagine `/etc/cron.daily/report` normally takes twenty minutes.

An administrator manually runs:

```bash
sudo run-parts /etc/cron.daily
```

while the scheduled daily run is still active.

Two copies may execute.

Use locking when duplicate execution is unsafe.

Example:

```bash
#!/bin/sh
set -eu

exec /usr/bin/flock -n /run/company-report.lock \
    /usr/local/libexec/company-report
```

Or:

```bash
#!/bin/sh
set -eu

/usr/bin/flock -n /run/company-report.lock \
    /usr/local/libexec/company-report
rc=$?

case "$rc" in
    0)
        ;;
    1)
        /usr/bin/logger -t company-report \
            "job skipped because another instance is running"
        ;;
    *)
        exit "$rc"
        ;;
esac
```

Locking semantics deserve their own chapter, but periodic directories do not eliminate concurrency concerns.

---

## Avoid using file existence alone as a lock

A weak pattern is:

```bash
if [ -e /tmp/job.lock ]; then
    exit 0
fi

touch /tmp/job.lock
do_work
rm -f /tmp/job.lock
```

Several problems exist:

```text
race condition between test and touch
stale lock if process crashes
unsafe /tmp interactions
predictable filename attacks
manual cleanup required
```

Kernel-backed advisory locking through `flock` is often preferable for local cron jobs.

This matters especially for root jobs.

A scheduler directory should never be treated as a replacement for correct process coordination.

---

## Test a periodic script outside cron first

Suppose you create:

```text
/etc/cron.daily/inventory-report
```

Validation should not begin by waiting until tomorrow.

Check syntax:

```bash
sudo sh -n /etc/cron.daily/inventory-report
```

if it is POSIX shell, or:

```bash
sudo bash -n /etc/cron.daily/inventory-report
```

for Bash.

Check discovery:

```bash
run-parts --test /etc/cron.daily | grep inventory-report
```

Run it directly:

```bash
sudo /etc/cron.daily/inventory-report
```

Run the directory in a controlled environment if appropriate:

```bash
sudo run-parts --test /etc/cron.daily
```

For a lab or isolated test directory:

```bash
sudo run-parts /tmp/test-cron-daily
```

Then verify outputs:

```bash
echo $?
journalctl --since "5 minutes ago"
```

The next scheduled window should confirm integration, not be the first functional test.

---

## Testing `/etc/cron.d/` jobs without changing system time

A file such as:

```text
/etc/cron.d/app-report
```

contains scheduling syntax, so `run-parts` is not the right test.

Instead, extract and execute the command under the intended identity.

Given:

```cron
15 3 * * * appuser /usr/local/libexec/app-report
```

test:

```bash
sudo -u appuser /usr/local/libexec/app-report
```

Then test with a reduced environment if relevant:

```bash
sudo -u appuser env -i \
    HOME=/var/lib/appuser \
    USER=appuser \
    LOGNAME=appuser \
    PATH=/usr/bin:/bin \
    SHELL=/bin/sh \
    /usr/local/libexec/app-report
```

Validate the crontab syntax using implementation-specific tools if available, or install into a test environment.

Never test scheduling by arbitrarily changing production system time.

That can disrupt:

```text
TLS validation
database timestamps
distributed systems
log ordering
authentication
monitoring
leases
cluster protocols
```

Time manipulation is not a safe cron debugging technique on a live server.

---

## Observe cron reloading behavior

Administrators sometimes ask whether they must restart cron after editing:

```text
/etc/crontab
/etc/cron.d/*
```

Many cron implementations monitor modification times and reload configuration automatically.

But the exact behavior is implementation-specific.

The safe operational habit is:

```bash
man cron
```

and inspect logs after editing.

A typical verification:

```bash
sudo touch /etc/cron.d/app-report
journalctl -u cron --since "1 minute ago"
```

or:

```bash
journalctl -u crond --since "1 minute ago"
```

may reveal reload messages.

Do not restart scheduling services unnecessarily on production systems without understanding the impact.

A cron daemon restart is usually small, but operational discipline favors verification over superstition.

---

## Syntax errors can invalidate individual entries

Consider:

```cron
SHELL=/bin/sh
PATH=/usr/bin:/bin

*/5 * * * * appuser /opt/app/check
not a valid cron line
0 2 * * * root /opt/app/backup
```

The handling of malformed files can vary.

Some parsers may reject invalid lines while retaining valid entries. Others may log errors and skip portions of the file.

The important action is to inspect logs immediately after modifying system cron files.

Use:

```bash
journalctl -u cron -n 100
```

or:

```bash
journalctl -u crond -n 100
```

and traditional log files where applicable:

```text
/var/log/syslog
/var/log/cron
```

Configuration changes should be treated like code deployment:

```text
write
validate
observe parser response
test execution
verify output
```

---

## Newline-at-end-of-file problems

Traditional text configuration parsers sometimes care about complete lines.

A malformed cron file that does not end with a newline may produce warnings or unexpected parsing behavior on some implementations.

Inspect:

```bash
tail -c 1 /etc/cron.d/app-report | od -An -t x1
```

A newline byte is:

```text
0a
```

A simple way to ensure a final newline is to edit the file normally with a text editor that preserves POSIX text-file conventions.

This is not the most common cron failure, but it is the kind of small formatting detail that appears during generated configuration or deployment automation.

---

## Windows line endings can break scripts

A cron script copied from Windows may contain CRLF line endings.

Inspect:

```bash
file /etc/cron.daily/company-job
```

Possible output:

```text
POSIX shell script, ASCII text executable, with CRLF line terminators
```

The shebang may effectively become:

```text
#!/bin/sh\r
```

which can cause an error such as:

```text
bad interpreter: No such file or directory
```

Fix:

```bash
sed -i 's/\r$//' /etc/cron.daily/company-job
```

or use:

```bash
dos2unix /etc/cron.daily/company-job
```

if available.

This problem is often misdiagnosed as a cron issue when it is really a text-format issue.

---

## The root filesystem is often the working directory

Many system cron invocations use:

```bash
cd /
```

before running periodic directories.

That creates predictable behavior:

```text
relative paths resolve from /
scripts do not inherit an administrator's shell directory
one job's path assumptions are easier to notice
```

Consider:

```bash
#!/bin/sh
echo "hello" > report.txt
```

If run from `/`, it creates:

```text
/report.txt
```

if permissions allow.

That is almost certainly not what the author intended.

Always use explicit paths:

```bash
echo "hello" > /var/lib/company/report.txt
```

or:

```bash
cd /var/lib/company
echo "hello" > report.txt
```

System automation should make file destinations obvious.

---

## Use `/usr/local/sbin` and `/usr/local/libexec` intentionally

For administrator-created system jobs, sensible locations include:

```text
/usr/local/sbin/
/usr/local/bin/
/usr/local/libexec/
```

A common layout:

```text
/etc/cron.d/company-backup
/usr/local/sbin/company-backup
/etc/company-backup/config
/var/lib/company-backup/
/var/log/company-backup/
```

This separates:

```text
schedule
executable
configuration
persistent runtime state
logs
```

from one another.

For a non-interactive helper that is not intended as a user command, a `libexec`-style location can communicate intent:

```text
/usr/local/libexec/company/
```

Linux distributions vary in exact filesystem conventions, but separation of concerns is valuable.

Avoid putting administrative scripts randomly in:

```text
/root/
/tmp/
/home/alice/Desktop/
```

simply because they were created there first.

---

## `/tmp` is a poor home for persistent cron scripts

This is unsafe and unreliable:

```cron
* * * * * root /tmp/backup.sh
```

Problems include:

```text
/tmp may be cleaned
other users can create files there
predictable path attacks
ownership mistakes
reboot cleanup
temporary-filesystem semantics
```

Use a controlled executable location instead:

```text
/usr/local/sbin/backup
```

Temporary data created by the job may use `/tmp` safely when written with secure primitives.

For example:

```bash
tmpfile=$(/usr/bin/mktemp)
trap 'rm -f "$tmpfile"' EXIT
```

But the scheduled program itself should not normally live in a world-writable temporary directory.

---

## Use `mktemp` instead of predictable temporary filenames

A periodic script might do:

```bash
echo "$data" > /tmp/report.tmp
mv /tmp/report.tmp /var/lib/report/report.txt
```

For privileged jobs, predictable `/tmp` paths can create symlink and race vulnerabilities.

Use:

```bash
tmpfile=$(/usr/bin/mktemp /tmp/report.XXXXXX)
trap 'rm -f "$tmpfile"' EXIT

generate_report > "$tmpfile"
/bin/mv "$tmpfile" /var/lib/report/report.txt
```

Even better, create the temporary file in the destination filesystem if atomic rename semantics matter:

```bash
tmpfile=$(/usr/bin/mktemp /var/lib/report/.report.XXXXXX)
```

This avoids cross-filesystem `mv` behavior.

Periodic root jobs deserve the same secure temporary-file design as any other privileged program.

---

## System jobs should be explicit about output

A cron directory job that writes nothing is easy to operate when successful, but failures must still be visible.

Example:

```bash
#!/bin/sh
set -eu

LOG_TAG=company-cleanup

if ! /usr/local/libexec/company/cleanup; then
    rc=$?
    /usr/bin/logger -p user.err -t "$LOG_TAG" \
        "cleanup failed rc=$rc"
    exit "$rc"
fi
```

Be careful with capturing `$?` after `!`, as discussed earlier.

A clearer form:

```bash
#!/bin/sh
set +e

/usr/local/libexec/company/cleanup
rc=$?

if [ "$rc" -ne 0 ]; then
    /usr/bin/logger -p user.err -t company-cleanup \
        "cleanup failed rc=$rc"
fi

exit "$rc"
```

Or simply let stderr and the exit code propagate if the parent cron environment has reliable monitoring.

The important design question is:

```text
Who notices when this fails?
```

A scheduled job without a failure-observation path is operationally incomplete.

---

## Do not confuse `/etc/cron.d/` with `/etc/cron.daily/`

The directory names are similar but their contents have different grammar.

`/etc/cron.d/example`:

```cron
0 2 * * * root /usr/local/sbin/example
```

`/etc/cron.daily/example`:

```bash
#!/bin/sh
exec /usr/local/sbin/example
```

Putting this inside `/etc/cron.daily/example` is wrong:

```cron
0 2 * * * root /usr/local/sbin/example
```

because `run-parts` treats the file as an executable program, not as a crontab.

Putting this in `/etc/cron.d/example` is also wrong:

```bash
#!/bin/sh
/usr/local/sbin/example
```

because the cron daemon expects crontab syntax there.

A useful rule:

```text
cron.d = schedule definitions
cron.daily = executable jobs
```

The same executable-directory principle applies to hourly, weekly, and monthly periodic directories.

---

## A practical system-job deployment

Suppose an application needs a root-controlled cleanup every six hours.

Create the executable:

```bash
sudo install -o root -g root -m 0755 \
    /dev/null /usr/local/sbin/company-cleanup
```

Edit:

```bash
sudo nano /usr/local/sbin/company-cleanup
```

Content:

```bash
#!/bin/sh
set -eu

PATH=/usr/bin:/bin
export PATH

umask 0027

TARGET=/var/lib/company/tmp

if [ ! -d "$TARGET" ]; then
    /usr/bin/logger -p user.err -t company-cleanup \
        "missing directory: $TARGET"
    exit 1
fi

/usr/bin/find "$TARGET" \
    -type f \
    -mtime +7 \
    -delete
```

Create the schedule:

```bash
sudo tee /etc/cron.d/company-cleanup >/dev/null <<'EOF'
SHELL=/bin/sh
PATH=/usr/bin:/bin

12 */6 * * * root /usr/local/sbin/company-cleanup
EOF
```

Set ownership:

```bash
sudo chown root:root /etc/cron.d/company-cleanup
sudo chmod 0644 /etc/cron.d/company-cleanup
```

Test executable syntax:

```bash
sudo sh -n /usr/local/sbin/company-cleanup
```

Run it manually:

```bash
sudo /usr/local/sbin/company-cleanup
```

Inspect recent cron logs:

```bash
journalctl -u cron --since "10 minutes ago"
```

This structure is clearer than putting the entire cleanup command into one long cron line.

---

## A practical daily-job deployment

Suppose exact clock time does not matter. It only needs to run once per daily maintenance cycle.

Create:

```bash
sudo tee /etc/cron.daily/company-report >/dev/null <<'EOF'
#!/bin/sh
set -eu

PATH=/usr/local/bin:/usr/bin:/bin
export PATH

umask 0027

cd /srv/company

exec /usr/local/bin/company-report \
    --config /etc/company/report.conf
EOF
```

Set:

```bash
sudo chown root:root /etc/cron.daily/company-report
sudo chmod 0755 /etc/cron.daily/company-report
```

Verify selection:

```bash
run-parts --test /etc/cron.daily | grep company-report
```

Test:

```bash
sudo /etc/cron.daily/company-report
```

Now determine who invokes the daily directory:

```bash
grep -R "/etc/cron.daily" \
    /etc/crontab \
    /etc/cron.d \
    /etc/anacrontab \
    2>/dev/null
```

and inspect timers:

```bash
systemctl list-timers --all
```

This final step prevents incorrect assumptions about actual execution time.

---

## Lab: distinguish user crontab and `/etc/cron.d/`

Create a script:

```bash
cat > /tmp/who-runs-me <<'EOF'
#!/bin/sh
{
    echo "date=$(/usr/bin/date -Is)"
    echo "uid=$(/usr/bin/id -u)"
    echo "user=$(/usr/bin/id -un)"
} >> /tmp/who-runs-me.log
EOF

chmod +x /tmp/who-runs-me
```

Add to your user crontab:

```cron
* * * * * /tmp/who-runs-me
```

Wait for execution:

```bash
cat /tmp/who-runs-me.log
```

Now remove the user entry and create, as root:

```bash
cat > /etc/cron.d/who-runs-me <<'EOF'
* * * * * root /tmp/who-runs-me
EOF

chown root:root /etc/cron.d/who-runs-me
chmod 0644 /etc/cron.d/who-runs-me
```

After another execution:

```bash
tail /tmp/who-runs-me.log
```

You should observe different execution identities.

Then change:

```cron
* * * * * nobody /tmp/who-runs-me
```

if the test account exists and can write the chosen destination. If not, choose a controlled test account and writable path.

The experiment demonstrates that the username field is part of system crontab semantics, not part of the command.

Clean up after the lab:

```bash
sudo rm -f /etc/cron.d/who-runs-me
rm -f /tmp/who-runs-me /tmp/who-runs-me.log
```

---

## Lab: prove that `/etc/cron.daily/` is not parsed as crontab syntax

Use a safe temporary directory:

```bash
mkdir -p /tmp/cron-daily-lab
```

Create:

```bash
cat > /tmp/cron-daily-lab/wrong-job <<'EOF'
* * * * * root echo hello
EOF

chmod +x /tmp/cron-daily-lab/wrong-job
```

Run:

```bash
run-parts /tmp/cron-daily-lab
```

The system attempts to execute the file as a program. It does not interpret the first five fields as a cron schedule.

Now replace it:

```bash
cat > /tmp/cron-daily-lab/correct-job <<'EOF'
#!/bin/sh
echo hello
EOF

chmod +x /tmp/cron-daily-lab/correct-job
rm -f /tmp/cron-daily-lab/wrong-job
```

Run:

```bash
run-parts /tmp/cron-daily-lab
```

Expected:

```text
hello
```

This simple lab makes the configuration-model difference concrete.

---

## Lab: identify why `backup.sh` is skipped

Create:

```bash
mkdir -p /tmp/parts-lab
```

Then:

```bash
cat > /tmp/parts-lab/backup.sh <<'EOF'
#!/bin/sh
echo "backup ran"
EOF

chmod +x /tmp/parts-lab/backup.sh
```

Check:

```bash
run-parts --test /tmp/parts-lab
```

If the implementation uses strict traditional naming rules, there may be no output.

Rename:

```bash
mv /tmp/parts-lab/backup.sh /tmp/parts-lab/backup
```

Check:

```bash
run-parts --test /tmp/parts-lab
```

Now:

```text
/tmp/parts-lab/backup
```

should appear.

Execute:

```bash
run-parts /tmp/parts-lab
```

Expected:

```text
backup ran
```

The lesson is important enough to repeat:

> File presence is not the same as scheduler eligibility.

---

## Lab: discover hidden privilege escalation

Create a fictional structure:

```bash
sudo mkdir -p /tmp/cron-security-lab/jobs
sudo tee /tmp/cron-security-lab/jobs/task >/dev/null <<'EOF'
#!/bin/sh
echo safe
EOF

sudo chmod 0755 /tmp/cron-security-lab/jobs/task
sudo chown root:root /tmp/cron-security-lab/jobs/task
```

Now inspect:

```bash
namei -l /tmp/cron-security-lab/jobs/task
```

Because `/tmp` is world-writable, this entire design would be inappropriate for a privileged scheduled script.

Even if:

```text
task
```

is owned by root, the surrounding path is not an appropriate root-controlled code location.

Repeat in:

```text
/usr/local/libexec/company/
```

with root-owned parent directories and compare the trust model.

The lab demonstrates why checking only:

```bash
ls -l script
```

is insufficient.

---

## Auditing all classic cron locations

A useful first-pass inventory:

```bash
sudo find \
    /etc/crontab \
    /etc/cron.d \
    /etc/cron.hourly \
    /etc/cron.daily \
    /etc/cron.weekly \
    /etc/cron.monthly \
    -maxdepth 2 \
    -type f \
    -ls 2>/dev/null
```

Then inspect system crontabs:

```bash
sudo cat /etc/crontab
```

and:

```bash
sudo grep -R -n -v '^[[:space:]]*#' /etc/cron.d 2>/dev/null
```

Inspect selected periodic jobs:

```bash
run-parts --test /etc/cron.hourly
run-parts --test /etc/cron.daily
run-parts --test /etc/cron.weekly
run-parts --test /etc/cron.monthly
```

List user crontabs in implementation-specific spool locations only if you understand the local layout and permissions.

Prefer:

```bash
crontab -l
sudo crontab -u USER -l
```

rather than editing spool files directly.

Finally check systemd:

```bash
systemctl list-timers --all
```

A modern scheduler audit is incomplete without both cron and systemd timer visibility.

---

## Find world-writable files in cron paths

A security audit may look for writable definitions and scripts.

For example:

```bash
sudo find /etc/cron.d \
    /etc/cron.hourly \
    /etc/cron.daily \
    /etc/cron.weekly \
    /etc/cron.monthly \
    -xdev \
    -type f \
    -perm -0002 \
    -ls 2>/dev/null
```

Group-writable files also deserve review:

```bash
sudo find /etc/cron.d \
    /etc/cron.hourly \
    /etc/cron.daily \
    /etc/cron.weekly \
    /etc/cron.monthly \
    -xdev \
    -type f \
    -perm -0020 \
    -ls 2>/dev/null
```

Group-writable is not always wrong. It depends on whether the group is trusted to control scheduled execution.

For every privileged cron command, inspect target path ownership.

A rough extraction of `/etc/cron.d/` commands is possible, but automated parsing is tricky because shell syntax is complex.

Manual review of privileged entries is often safer than over-trusting a simplistic parser.

---

## A secure review question set

When examining a system cron job, ask:

```text
Who can modify the schedule?
Which user executes the command?
Who can modify the executable?
Who can modify its parent directories?
Does it source other files?
Does it load plugins or modules from writable locations?
Does it rely on PATH lookup?
Does it use temporary files safely?
Does it write to a mounted filesystem?
Can two instances overlap?
Where do stdout and stderr go?
Who monitors failures?
Is the job duplicated in systemd?
Is it package-owned?
Will upgrades overwrite local changes?
```

These questions reveal much more than reading the five scheduling fields.

---

## Choosing between `/etc/crontab`, `/etc/cron.d/`, and periodic directories

Use `/etc/crontab` when maintaining a small amount of global local scheduling and the system already uses it clearly.

Use `/etc/cron.d/` when a separate service, application, or package should own a self-contained schedule file.

Use `/etc/cron.hourly/`, `/etc/cron.daily/`, `/etc/cron.weekly/`, or `/etc/cron.monthly/` when:

```text
exact execution time is not important
the task naturally belongs to a frequency class
the distribution's periodic-job infrastructure is appropriate
```

Use a direct cron schedule or another scheduler when the requirement is precise.

For example:

```text
"once per day" -> cron.daily may be appropriate

"every day exactly at 02:17" -> explicit cron entry is clearer

"run after boot if yesterday's job was missed" -> anacron/systemd timer may be better

"run 15 minutes after another service becomes active" -> systemd dependency model may be better
```

The storage location should match the semantics of the job.

---

## A clean directory layout for local administration

A maintainable local automation setup can look like:

```text
/etc/cron.d/
└── company-maintenance

/etc/company/
├── backup.conf
└── report.conf

/usr/local/libexec/company/
├── backup
├── cleanup
└── report

/var/lib/company/
└── scheduler-state/

/var/log/company/
├── backup.log
├── cleanup.log
└── report.log
```

The schedule file:

```cron
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

10 1 * * * root /usr/local/libexec/company/backup
40 2 * * * root /usr/local/libexec/company/cleanup
15 6 * * * report /usr/local/libexec/company/report
```

This makes ownership and responsibility visible.

Compare that with:

```text
/root/x.sh
/tmp/doit
/home/dev/final-final-backup.sh
/etc/cron.d/test2
```

Production reliability often begins with boring filesystem organization.

---

## Troubleshooting a job in `/etc/cron.d/`

Suppose:

```text
/etc/cron.d/api-report
```

contains:

```cron
0 * * * * api /opt/api/bin/report
```

but nothing appears to happen.

Check the file itself:

```bash
sudo stat /etc/cron.d/api-report
sudo cat -A /etc/cron.d/api-report
```

`cat -A` can reveal line-ending problems and missing newlines.

Check account existence:

```bash
getent passwd api
```

Check executable:

```bash
sudo -u api test -x /opt/api/bin/report
echo $?
```

Check path traversal:

```bash
namei -l /opt/api/bin/report
```

Run under the account:

```bash
sudo -u api /opt/api/bin/report
```

Run with a reduced environment:

```bash
sudo -u api env -i \
    HOME="$(getent passwd api | cut -d: -f6)" \
    USER=api \
    LOGNAME=api \
    PATH=/usr/bin:/bin \
    SHELL=/bin/sh \
    /opt/api/bin/report
```

Inspect cron logs:

```bash
journalctl -u cron --since "2 hours ago"
```

or:

```bash
journalctl -u crond --since "2 hours ago"
```

Check duplicates:

```bash
grep -R -n "/opt/api/bin/report" /etc/cron* 2>/dev/null
systemctl list-timers --all | grep -i api
```

This workflow moves from configuration parsing to runtime execution systematically.

---

## Troubleshooting a job in `/etc/cron.daily/`

Suppose:

```text
/etc/cron.daily/api-cleanup
```

does not appear to run.

Check:

```bash
ls -l /etc/cron.daily/api-cleanup
```

Check file type:

```bash
file /etc/cron.daily/api-cleanup
```

Check shebang:

```bash
head -n 1 /etc/cron.daily/api-cleanup
```

Check filename eligibility:

```bash
run-parts --test /etc/cron.daily | grep api-cleanup
```

Check direct execution:

```bash
sudo /etc/cron.daily/api-cleanup
echo $?
```

Determine who invokes the directory:

```bash
grep -R "/etc/cron.daily" \
    /etc/crontab \
    /etc/cron.d \
    /etc/anacrontab \
    2>/dev/null
```

Check timers:

```bash
systemctl list-timers --all
```

Inspect logs around the expected time.

The troubleshooting logic differs from `/etc/cron.d/` because one is parsed cron syntax and the other is executable discovery.

---

## System jobs are part of configuration management

On a single test VM, manually creating:

```text
/etc/cron.d/foo
```

is simple.

Across fifty servers, manual configuration becomes drift.

Use configuration management or deployment automation to control:

```text
file contents
owner
group
mode
script checksum
package dependency
service account
log directory
configuration file
scheduler definition
```

A declarative deployment should be able to answer:

```text
Which version of the job is installed?
Which servers run it?
Who owns it?
When was it changed?
```

Cron is not exempt from infrastructure-as-code discipline.

A small cron line can perform highly privileged work. It deserves the same review standards as a service unit or application deployment.

---

## Cron directories are not application queues

A periodic directory is useful for independent maintenance tasks.

It is not a robust replacement for:

```text
job queue
workflow engine
distributed scheduler
retry system
event processor
```

If an application requires:

```text
retry with exponential backoff
per-job state
distributed locking
dependency graph
concurrency limits
exactly-once semantics
per-task history
manual replay
```

traditional cron directories are too small a model.

Cron can trigger the application's scheduler:

```cron
* * * * * app /srv/app/bin/run-scheduler
```

but the application should own application-level scheduling semantics.

This separation keeps cron responsible for operating-system timing rather than business workflow.

---

## Final perspective

Linux system cron is not one file and one daemon configuration.

It is an ecosystem of schedule definitions, executable directories, package-managed jobs, helper programs such as `run-parts`, optional anacron behavior, and increasingly systemd timers.

The most important distinctions are structural.

`/etc/crontab` and `/etc/cron.d/` contain schedule definitions.

Their entries generally include an explicit user field because they can launch jobs under multiple identities.

Periodic directories such as `/etc/cron.daily/` contain executables, not crontab expressions.

A helper such as `run-parts` selects and launches those executables.

File names matter.

Execute bits matter.

Ownership matters.

Parent-directory permissions matter.

The scheduler that actually invokes the directory matters.

A system job is therefore not fully described by:

```text
when does it run?
```

A proper description also includes:

```text
where is its schedule stored?
which parser reads it?
which user launches it?
which executable is trusted?
which filesystem paths can influence it?
which package owns it?
which runner selects it?
where does output go?
what happens if it fails?
what happens if it runs twice?
```

Once those questions are answered explicitly, cron directories stop being mysterious.

They become what they really are: a small, predictable filesystem-based interface between Linux administration and scheduled process execution.
