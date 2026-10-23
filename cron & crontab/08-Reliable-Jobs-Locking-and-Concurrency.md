# Reliable Cron Jobs, Locking, and Concurrency

A cron expression can be perfectly correct while the job is operationally unsafe.

Consider:

```cron
* * * * * /opt/jobs/sync.sh
```

The schedule says:

```text
start once per minute
```

It does not say:

```text
wait for the previous run to finish
```

If `sync.sh` takes four minutes, the system can have several copies running at the same time.

At 12:00:

```text
sync.sh PID 4100
```

At 12:01:

```text
sync.sh PID 4100
sync.sh PID 4172
```

At 12:02:

```text
sync.sh PID 4100
sync.sh PID 4172
sync.sh PID 4251
```

At 12:03:

```text
sync.sh PID 4100
sync.sh PID 4172
sync.sh PID 4251
sync.sh PID 4338
```

Cron is behaving correctly.

The scheduler is doing exactly what the schedule requested.

The problem is concurrency.

A reliable scheduled job must define what should happen when a new run becomes eligible while a previous run is still active.

Possible policies include:

```text
allow overlap
skip the new run
wait until the previous run finishes
terminate the old run
serialize all runs
queue work elsewhere
use distributed coordination
```

There is no universally correct policy. It depends on the job.

A monitoring probe may safely overlap.

A database migration must not.

A report generator may be idempotent.

A backup process may corrupt its destination if two copies write simultaneously.

Cron itself provides timing. Concurrency control belongs to the job design.

---

## Overlap is normal cron behavior

A cron daemon does not generally maintain a per-entry state machine such as:

```text
RUNNING
WAITING
FINISHED
```

for traditional jobs.

It evaluates the schedule and launches a command when the time matches.

That means:

```cron
*/5 * * * * /opt/jobs/task
```

can start at:

```text
12:00
12:05
12:10
12:15
```

even if the 12:00 run is still active at 12:05.

This is not a bug.

Cron does not automatically infer that two invocations belong to the same logical task.

The operating system sees only processes.

The application must define whether concurrent processes are acceptable.

---

## Measure runtime before choosing a schedule

A surprisingly common mistake is choosing a frequency before knowing how long the job takes.

Test:

```bash
/usr/bin/time -p /opt/jobs/task
```

Example:

```text
real 223.47
user 18.21
sys 6.32
```

A runtime of approximately 223 seconds means a schedule of:

```cron
*/5 * * * * ...
```

normally has only about 77 seconds of spare time.

Any temporary slowdown can create overlap.

Record runtime over many executions rather than relying on one measurement.

A wrapper can log duration:

```bash
#!/bin/sh

start=$(/usr/bin/date +%s)

/opt/jobs/task
rc=$?

end=$(/usr/bin/date +%s)
duration=$((end - start))

/usr/bin/logger -t task \
    "rc=$rc duration_seconds=$duration"

exit "$rc"
```

After several days:

```bash
journalctl -t task
```

may show:

```text
duration_seconds=190
duration_seconds=205
duration_seconds=221
duration_seconds=318
duration_seconds=184
```

The 318-second execution is enough to overlap a five-minute interval.

Scheduling should be based on worst-case behavior, not average behavior alone.

---

## Concurrency is not always bad

Some tasks are intentionally parallel.

Suppose cron launches independent health checks:

```cron
* * * * * /opt/checks/check-region-a
* * * * * /opt/checks/check-region-b
* * * * * /opt/checks/check-region-c
```

They can safely run simultaneously because they do not share mutable state.

Likewise, a job may process immutable input:

```text
read object
calculate checksum
write uniquely named result
```

Multiple copies can be safe.

The question is not:

```text
Are two processes running?
```

The real question is:

```text
Can two executions interfere with each other?
```

Interference can occur through:

```text
same output file
same temporary file
same database row
same remote API resource
same lockless counter
same backup destination
same package manager
same filesystem tree
same service restart
same queue
same device
```

Concurrency must be analyzed around shared state.

---

## A simple overlapping-job failure

Consider:

```bash
#!/bin/sh

echo "starting"

/usr/bin/curl \
    --fail \
    --silent \
    https://api.example.com/data \
    > /var/lib/app/data.json

echo "done"
```

Cron:

```cron
* * * * * /opt/jobs/fetch-data
```

Suppose the request normally takes five seconds but one day takes 80 seconds.

At 12:00:

```text
PID 5000 opens data.json with >
```

The shell truncates the file.

At 12:01:

```text
PID 5100 opens the same data.json with >
```

The shell truncates it again while PID 5000 may still be writing.

Now the final file depends on timing.

The two processes race over:

```text
/var/lib/app/data.json
```

This is a concurrency bug.

A safer design writes a private temporary file and performs an atomic rename:

```bash
#!/bin/sh
set -eu

DIR=/var/lib/app
tmp=$(/usr/bin/mktemp "$DIR/.data.XXXXXX")

cleanup() {
    rm -f -- "$tmp"
}

trap cleanup EXIT

/usr/bin/curl \
    --fail \
    --silent \
    --show-error \
    https://api.example.com/data \
    > "$tmp"

/bin/mv "$tmp" "$DIR/data.json"

trap - EXIT
```

This removes partial-file exposure.

It does not yet solve the question of which concurrent run should win.

Atomic file replacement and process serialization are related but different problems.

---

## Atomicity versus locking

Atomicity means an operation appears indivisible to observers.

Locking means participants coordinate access to a shared resource.

Suppose two processes each write a new file and atomically rename it to:

```text
data.json
```

Readers never observe a half-written file.

That is good.

But if the older process finishes after the newer process, the older data may overwrite the newer data.

Example:

```text
12:00 run starts
12:01 run starts
12:01 run finishes -> data from 12:01 installed
12:00 run finishes -> older data replaces newer data
```

Every rename was atomic.

The final logical state is still wrong.

A lock can serialize the runs so only one process updates the file at a time.

This distinction is important:

```text
atomicity protects the shape of an update
locking protects coordination between actors
```

Many reliable jobs need both.

---

## Race conditions

A race condition exists when program correctness depends on the unpredictable timing of concurrent events.

Classic example:

```bash
if [ ! -e /tmp/job.running ]; then
    touch /tmp/job.running
    do_work
    rm -f /tmp/job.running
fi
```

Two processes can execute:

```text
Process A: test -> no file
Process B: test -> no file
Process A: touch
Process B: touch
Process A: do_work
Process B: do_work
```

Both enter the critical section.

The operation:

```text
check whether file exists
then create it
```

is not atomic.

This is why naive lock files are unreliable.

A correct lock primitive needs an atomic acquisition operation.

---

## `flock` is the standard local locking tool

Linux commonly provides:

```bash
flock
```

from util-linux.

Check:

```bash
command -v flock
```

Possible output:

```text
/usr/bin/flock
```

Basic usage:

```bash
flock LOCKFILE COMMAND
```

Example:

```bash
flock /run/myjob.lock /opt/jobs/myjob
```

The command acquires an advisory lock associated with the lock file before launching the job.

A cron entry:

```cron
* * * * * /usr/bin/flock /run/myjob.lock /opt/jobs/myjob
```

has an important default behavior:

```text
if another process holds the lock,
wait
```

That may or may not be what you want.

For cron jobs, non-blocking mode is often more useful.

---

## Non-blocking locking with `flock -n`

Use:

```bash
flock -n /run/myjob.lock /opt/jobs/myjob
```

`-n` means:

```text
do not wait
```

If the lock is already held, the new invocation exits immediately.

Cron:

```cron
* * * * * /usr/bin/flock -n /run/myjob.lock /opt/jobs/myjob
```

Now:

```text
12:00 job starts and gets lock
12:01 cron launches another instance
12:01 flock cannot acquire lock
12:01 new instance exits
12:02 same if old job still runs
```

This implements:

```text
skip if already running
```

That policy is appropriate for many periodic maintenance jobs.

---

## Waiting versus skipping

Compare:

```bash
flock /run/job.lock /opt/jobs/job
```

with:

```bash
flock -n /run/job.lock /opt/jobs/job
```

Blocking behavior:

```text
run A holds lock
run B waits
run C waits
run D waits
```

Once A finishes:

```text
one waiter proceeds
```

This can turn cron into an accidental queue.

If the job is consistently slower than its schedule interval, waiting processes accumulate.

Check:

```bash
ps -ef | grep '[f]lock.*job.lock'
```

You may see many blocked invocations.

Non-blocking behavior avoids backlog:

```text
run A holds lock
run B skips
run C skips
run D skips
```

For recurring state reconciliation, skipping is often correct because the next successful run can catch up.

For every-event processing, skipping may lose required work.

Concurrency policy must match job semantics.

---

## A dangerous backlog example

Cron:

```cron
* * * * * /usr/bin/flock /run/report.lock /opt/jobs/report
```

Runtime:

```text
10 minutes
```

Because `flock` waits, after ten minutes there may be approximately ten scheduled processes:

```text
one running
nine waiting
```

When the first finishes, the next begins.

Meanwhile cron keeps adding more.

The system is permanently behind.

This is not serialization in a healthy sense.

It is queue growth.

If the task represents:

```text
rebuild current report from current database state
```

then old queued executions are useless.

Use:

```bash
flock -n
```

so missed intervals are discarded.

If every run represents distinct required work, use a real work queue rather than cron-generated process backlog.

---

## Time-limited lock waiting

Sometimes waiting briefly is appropriate.

GNU/util-linux `flock` supports a timeout option on many systems:

```bash
flock -w 30 /run/job.lock /opt/jobs/job
```

This means:

```text
wait up to 30 seconds for the lock
```

If it becomes available, run.

Otherwise exit.

Example cron:

```cron
*/5 * * * * /usr/bin/flock -w 20 /run/reconcile.lock /opt/jobs/reconcile
```

This allows small timing overlap without creating long queues.

Check exact options:

```bash
flock --help
man flock
```

because installed versions can differ.

---

## Lock location matters

Common local lock locations include:

```text
/run/
/run/lock/
/var/lock/
```

Modern Linux systems often mount `/run` as tmpfs.

That means lock artifacts disappear at reboot.

For an active process lock, that is usually desirable.

Example:

```text
/run/company-backup.lock
```

Do not put privileged lock files in a world-writable location without understanding the security consequences.

Avoid:

```text
/tmp/root-backup.lock
```

for a root job if an unprivileged user can manipulate the path.

A safe root job can use:

```bash
install -d \
    -o root \
    -g root \
    -m 0755 \
    /run/company
```

then:

```text
/run/company/backup.lock
```

For an unprivileged service account:

```bash
install -d \
    -o appuser \
    -g appuser \
    -m 0750 \
    /run/appuser
```

If `/run` is recreated on boot, system initialization must recreate the subdirectory.

Systemd tmpfiles is one way to manage runtime directories on systems using systemd.

---

## A lock file is not necessarily a state file

With `flock`, the lock pathname often persists as an empty file even when no process holds the lock.

Example:

```bash
touch /tmp/example.lock
flock /tmp/example.lock sleep 30
```

In another terminal:

```bash
ls -l /tmp/example.lock
```

The file exists.

After `sleep` exits, the file may still exist.

But the lock is released.

Therefore:

```text
lock file exists
```

does not mean:

```text
job is running
```

for `flock`.

The lock state is maintained by the kernel against an open file description or descriptor, not by file existence alone.

This eliminates the classic stale-file problem.

---

## Why `flock` does not need manual lock deletion

A typical `flock` lock is released when the process or file descriptor holding it closes.

If the process:

```text
exits normally
crashes
receives SIGKILL
```

the kernel closes the descriptor and releases the lock.

That is one of the main advantages over:

```bash
touch /tmp/job.lock
...
rm /tmp/job.lock
```

The manual scheme can leave a stale file after a crash.

Kernel locks follow process lifetime.

This makes `flock` particularly suitable for cron.

---

## Advisory locking

`flock` normally provides advisory locks.

Advisory means:

```text
cooperating processes must honor the lock
```

If process A locks:

```text
/var/lib/app/state
```

process B can still open and modify the file if B does not use compatible locking.

The kernel does not automatically prevent every ordinary read/write based on an advisory lock.

Therefore all participants sharing the protected resource must follow the same locking protocol.

If only the cron job writes the state, this is straightforward.

If several independent applications modify the same data, coordination must be designed across all of them.

---

## Lock the operation, not an unrelated pathname by accident

This:

```bash
flock /run/job.lock /opt/jobs/job
```

uses:

```text
/run/job.lock
```

as the coordination object.

All competing instances must use exactly the same lock identity.

If one script uses:

```text
/run/job.lock
```

and another uses:

```text
/var/lock/job.lock
```

they do not coordinate.

Similarly, two separate host containers may see different filesystems and therefore different lock objects even when path strings look identical.

A locking scheme only works if every actor reaches the same synchronization primitive.

---

## File-descriptor style `flock`

Instead of wrapping the command:

```bash
flock -n /run/job.lock /opt/jobs/job
```

a script can manage a lock file descriptor.

Example:

```bash
#!/usr/bin/env bash
set -euo pipefail

exec 9>/run/company-job.lock

if ! flock -n 9; then
    echo "another instance is running" >&2
    exit 0
fi

do_work
```

Here:

```text
FD 9
```

is opened on the lock file.

`flock -n 9` locks the file associated with that descriptor.

The lock remains held as long as the descriptor remains open.

When the script exits, the kernel closes it and releases the lock.

This style is useful when the script needs explicit behavior on lock contention.

---

## Use a dedicated exit code for skipped runs

By default, lock acquisition failure may produce an exit status that monitoring interprets as failure.

That may be appropriate.

But if:

```text
another instance is running
```

is a normal skip condition, distinguish it from real failure.

One pattern:

```bash
#!/usr/bin/env bash
set -u

exec 9>/run/report.lock

if ! flock -n 9; then
    logger -t report-job \
        "event=skip reason=lock_held"
    exit 0
fi

run_report
```

Now skipping is considered normal.

Another policy is to exit with a dedicated status:

```bash
exit 75
```

and teach monitoring that:

```text
75 = temporary skip
```

The exact code is a local convention.

The important point is to classify outcomes deliberately:

```text
success
skip
failure
timeout
```

---

## `flock -E` can choose the conflict exit code

Some util-linux versions support:

```bash
flock -E 75 -n /run/job.lock /opt/jobs/job
```

Then lock conflict returns:

```text
75
```

instead of the default conflict code.

Check:

```bash
flock --help
```

before relying on the option.

This is useful for wrappers that distinguish:

```text
could not acquire lock
```

from:

```text
job itself failed
```

Example:

```bash
/usr/bin/flock \
    -E 75 \
    -n \
    /run/job.lock \
    /opt/jobs/job

rc=$?

case "$rc" in
    0)
        echo success
        ;;
    75)
        echo skipped
        ;;
    *)
        echo failed >&2
        ;;
esac
```

---

## Locking inside the script is often clearer

Cron:

```cron
* * * * * /opt/jobs/report
```

Script:

```bash
#!/usr/bin/env bash
set -euo pipefail

LOCK=/run/report.lock

exec 9>"$LOCK"

if ! flock -n 9; then
    logger -t report \
        "event=skip reason=already_running"
    exit 0
fi

logger -t report "event=start"

do_work

logger -t report "event=finish rc=0"
```

Advantages:

```text
locking policy is version-controlled with the job
manual executions use the same lock
systemd or operator runs also use the same lock
logging can distinguish lock conflict
cron line stays simple
```

If the lock exists only in the crontab wrapper, someone running:

```bash
/opt/jobs/report
```

manually bypasses the lock.

For critical serialization, locking inside the program is stronger.

---

## But avoid double-lock confusion

Do not casually do both:

```cron
* * * * * flock -n /run/report.lock /opt/jobs/report
```

and inside:

```bash
flock -n /run/report.lock ...
```

Depending on descriptor semantics and lock ownership, the nested lock can fail against the lock already held by the wrapper or behave in surprising ways.

Choose one clear locking layer.

Usually:

```text
inside the application/wrapper
```

is easier to reuse correctly.

---

## `mkdir` can be used as an atomic lock primitive

On Unix filesystems, creating a directory is atomic with respect to the pathname.

A simple lock:

```bash
if mkdir /run/myjob.lockdir 2>/dev/null; then
    echo "lock acquired"
else
    echo "already locked"
fi
```

Two concurrent processes cannot both successfully create the same directory.

This is stronger than:

```bash
test -e lockfile
touch lockfile
```

because acquisition occurs in one atomic filesystem operation.

But stale-state handling remains.

If the process crashes before:

```bash
rmdir /run/myjob.lockdir
```

the directory remains.

That requires a recovery strategy.

`flock` is usually simpler for local Linux process locking.

---

## Why PID files are not reliable locks by themselves

A common pattern:

```bash
echo $$ > /run/job.pid
```

and later:

```bash
if kill -0 "$(cat /run/job.pid)"; then
    echo running
fi
```

Problems include:

```text
stale PID file after crash
PID reuse
race between checking and writing
process identity mismatch
file tampering
```

Suppose PID 1234 belonged to yesterday's job.

The machine later assigns PID 1234 to:

```text
sshd
```

A naive PID-file check reports:

```text
job is still running
```

even though it is not.

PID files can provide useful metadata.

They should not be mistaken for a strong locking primitive.

---

## PID reuse

Linux process IDs are finite integers.

After processes exit, their PIDs eventually become available for reuse.

Check the maximum:

```bash
cat /proc/sys/kernel/pid_max
```

A stale file containing:

```text
31842
```

can eventually point to an unrelated process.

If process identity matters, compare more than PID.

Possible additional data:

```text
process start time
command line
executable
boot ID
```

But by the time a locking mechanism requires this much recovery logic, `flock` is usually a better local solution.

---

## Lock ownership and deletion races

Suppose process A uses a file lock on:

```text
/run/job.lock
```

and another process deletes and recreates that pathname.

Advisory locking is associated with file objects/descriptors, not simply visible path strings.

Now different processes may end up locking different inodes while both paths appear to be:

```text
/run/job.lock
```

Therefore, do not routinely delete active `flock` lock files.

Create the lock file in a protected directory and leave it in place.

The kernel-managed lock state is what matters.

---

## Inspect a lock file's inode

Example:

```bash
stat /run/job.lock
```

shows:

```text
Inode: 123456
```

A process holding the file open can be inspected:

```bash
lsof /run/job.lock
```

Possible:

```text
COMMAND   PID USER FD TYPE DEVICE SIZE/OFF NODE NAME
job      5120 app  9w REG  0,25        0 123456 /run/job.lock
```

This is useful during diagnosis.

Another tool:

```bash
lslocks
```

can list current locks on many Linux systems.

Check:

```bash
command -v lslocks
```

Then:

```bash
lslocks
```

Output may include:

```text
COMMAND PID TYPE SIZE MODE M START END PATH
job    5120 FLOCK   0B WRITE 0     0   0 /run/job.lock
```

Exact columns depend on util-linux version.

---

## `lslocks` is valuable for lock debugging

Filter:

```bash
lslocks | grep job.lock
```

or use output-selection options supported by the installed version.

This answers:

```text
Is there an active kernel lock?
Which PID owns it?
What path/inode is involved?
```

That is much stronger than:

```bash
ls -l /run/job.lock
```

because a lock file can exist with no active holder.

---

## A locking lab

Create:

```bash
cat > /tmp/lock-lab.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

exec 9>/tmp/lock-lab.lock

if ! flock -n 9; then
    echo "[$$] lock busy"
    exit 0
fi

echo "[$$] acquired"
sleep 20
echo "[$$] releasing"
EOF

chmod +x /tmp/lock-lab.sh
```

In terminal one:

```bash
/tmp/lock-lab.sh
```

Immediately in terminal two:

```bash
/tmp/lock-lab.sh
```

Expected:

```text
[PID] lock busy
```

After the first process exits, run again:

```bash
/tmp/lock-lab.sh
```

It acquires the lock.

The file:

```text
/tmp/lock-lab.lock
```

may remain the entire time.

This lab demonstrates kernel lock lifetime clearly.

---

## A blocking-lock lab

Change:

```bash
flock -n 9
```

to:

```bash
flock 9
```

Now start three copies quickly:

```bash
/tmp/lock-lab.sh &
/tmp/lock-lab.sh &
/tmp/lock-lab.sh &
```

Inspect:

```bash
jobs
```

and:

```bash
ps -ef | grep '[l]ock-lab'
```

Only one should be inside the critical section at a time.

The others wait.

This demonstrates serialization.

It also demonstrates why blocking locks can create a backlog under cron.

---

## Use locking around the critical section, not necessarily the whole process

Suppose a job has three phases:

```text
download input
update shared database
upload report
```

Only database update must be serialized.

Instead of locking the entire job:

```bash
flock lockfile whole-job
```

the application can lock only the critical section.

Conceptually:

```bash
download_input

acquire_lock
update_shared_state
release_lock

upload_report
```

This allows independent expensive work to happen concurrently while protecting shared state.

However, partial locking makes correctness reasoning more complex.

For cron shell scripts, whole-job locking is often simpler and safer unless performance requires finer granularity.

---

## Idempotency

An idempotent operation can be repeated without changing the final result beyond the first successful application.

Example:

```bash
mkdir -p /var/lib/app/cache
```

Running it once or ten times leaves the same directory present.

A non-idempotent operation:

```bash
echo "$amount" >> /var/lib/app/balance-events
```

adds another event every time.

Idempotency matters because scheduled systems experience:

```text
retries
manual reruns
duplicate scheduler entries
server restart
operator mistakes
network uncertainty
partial failures
```

A job that is safe to repeat is much easier to operate.

Locking prevents concurrent duplicate execution.

Idempotency reduces damage if duplication still occurs.

These mechanisms complement each other.

---

## Idempotent report generation

Weak:

```bash
cat new-data >> /var/lib/report/daily.csv
```

Every rerun duplicates rows.

Stronger:

```bash
tmp=$(mktemp /var/lib/report/.daily.XXXXXX)

generate_complete_report > "$tmp"
mv "$tmp" /var/lib/report/daily.csv
```

Every successful run replaces the complete report.

The operation becomes naturally repeatable.

This architecture is often better than complicated duplicate-detection logic.

---

## Idempotent database work

Suppose a daily job inserts one summary row:

```sql
INSERT INTO daily_summary (date, total)
VALUES ('2026-09-13', 1234);
```

A rerun creates a duplicate unless the schema prevents it.

A stronger schema:

```sql
CREATE UNIQUE INDEX
ON daily_summary(date);
```

Then use database-specific upsert behavior.

Conceptually:

```sql
INSERT ...
ON CONFLICT (date)
DO UPDATE ...
```

Now correctness is enforced at the database layer.

This is much stronger than trusting cron never to run twice.

A scheduler should not be the only guardian of data integrity.

---

## Database constraints are concurrency controls

Two concurrent processes can both execute:

```text
check whether row exists
if not -> insert
```

and race.

Application-level check:

```text
SELECT
then INSERT
```

is not sufficient by itself.

A unique constraint gives the database authority to reject the duplicate.

Use:

```text
primary keys
unique constraints
transactions
row locks
advisory locks
atomic updates
```

for shared database state.

Cron-level `flock` only coordinates processes on one host.

The database can coordinate all clients that connect to it.

---

## Single-host lock versus distributed lock

`flock` works against a file visible to the local kernel/filesystem.

Suppose the same cron job runs on three servers:

```text
app-01
app-02
app-03
```

Each uses:

```text
/run/report.lock
```

Those are three separate lock files on three separate machines.

All three jobs can run simultaneously.

Local lock:

```text
protects against duplicates on one coordination domain
```

It does not automatically protect across a fleet.

If only one node should perform the job globally, use distributed coordination.

Possible coordination sources include:

```text
database advisory lock
distributed key-value store
orchestrator leader election
cloud scheduler
queue worker ownership
distributed lock service
```

The correct design depends on infrastructure.

---

## Shared filesystem locks are not automatically distributed locks

It may be tempting to place:

```text
/shared/report.lock
```

on NFS and assume `flock` now coordinates all servers.

Lock semantics on network filesystems depend on:

```text
filesystem type
mount options
server support
kernel client implementation
failure recovery
network partitions
```

Do not build critical distributed coordination on a shared file lock without validating the exact filesystem semantics.

A database or purpose-built coordination system is often easier to reason about.

---

## Database advisory locks

Many databases provide session-level advisory locks.

For PostgreSQL, applications can use advisory locking functions such as:

```sql
pg_try_advisory_lock(...)
```

Conceptually:

```text
attempt global lock inside database
if lock unavailable -> skip
if acquired -> perform work
connection closes -> lock released
```

This is useful when multiple application servers schedule the same logical job.

The lock lives in the same centralized system all servers reach.

A job wrapper might call application code that performs:

```text
begin connection
try advisory lock
run task
release lock
```

This is generally preferable to implementing distributed locking manually in shell.

The database-specific semantics should be studied carefully before production use.

---

## Redis distributed locks require careful design

Redis is often used for distributed locks.

A simplistic pattern:

```text
SET lock-key token NX EX 300
```

is not enough by itself for every workload.

Questions include:

```text
What if the job lasts more than 300 seconds?
What if the owner pauses?
What if the lock expires while work continues?
How is ownership verified on release?
What happens during network partitions?
What consistency guarantees are required?
```

Distributed locking is substantially harder than local `flock`.

If the workload is important, use well-understood framework primitives or a central job queue rather than inventing a lock protocol casually.

---

## Lease expiration can allow concurrent owners

Suppose:

```text
worker A acquires 5-minute lease
worker A pauses for 6 minutes
lease expires
worker B acquires lock
worker A resumes
```

Now both believe they can operate.

This is a fundamental difference between:

```text
process-lifetime local kernel lock
```

and:

```text
time-limited distributed lease
```

Some systems use fencing tokens so downstream resources can reject stale owners.

This is beyond ordinary cron administration, but important for understanding why:

```text
just use Redis with expiry
```

is not always a complete distributed concurrency solution.

---

## Cron on multiple application servers

A common deployment mistake:

```text
three web servers
same crontab baked into all images
same nightly invoice generation
```

At midnight:

```text
all three generate invoices
```

Possible consequences:

```text
duplicate invoices
duplicate emails
double billing
duplicate report rows
```

Solutions include:

```text
run scheduler on one designated node
application-level distributed lock
database constraint
queue architecture
orchestrator CronJob with correct concurrency policy
external scheduler
```

Relying on:

```text
we normally have one server
```

is fragile if autoscaling or disaster recovery later adds another node.

---

## Designate one scheduler node explicitly

One simple architecture:

```text
web-01 -> normal app only
web-02 -> normal app only
scheduler-01 -> cron jobs
```

Configuration management installs scheduler entries only on:

```text
scheduler-01
```

This is operationally simple.

But it introduces a single scheduler dependency.

If `scheduler-01` is unavailable, jobs do not run.

For many small systems this tradeoff is acceptable.

For highly available workloads, use a scheduler with built-in clustering or distributed ownership.

---

## Application scheduler plus cron trigger

Frameworks often use cron only as a one-minute trigger:

```cron
* * * * * cd /srv/app && php artisan schedule:run
```

The application framework then determines which internal tasks are due.

Concurrency concerns now exist at two levels:

```text
cron trigger overlap
application task overlap
```

A framework may offer features similar to:

```text
without overlapping
run on one server
background execution
mutexes
```

Use framework-native coordination when appropriate.

Do not assume the outer cron trigger alone prevents duplicate internal jobs.

---

## Lock scope should match logical identity

Suppose a job processes one customer:

```bash
process-customer CUSTOMER_ID
```

A single global lock:

```text
/run/customer-processing.lock
```

serializes every customer.

That may be unnecessarily restrictive.

Use per-resource locks:

```text
/run/company/customer-123.lock
/run/company/customer-456.lock
```

Then:

```text
same customer -> serialized
different customers -> parallel
```

Lock key design determines concurrency granularity.

The same idea appears in databases and distributed locks.

---

## Validate lock keys

Do not build lock paths directly from untrusted data:

```bash
LOCK="/run/jobs/$CUSTOMER_ID.lock"
```

if `CUSTOMER_ID` can contain:

```text
../../
slashes
control characters
```

Validate:

```bash
case "$CUSTOMER_ID" in
    *[!A-Za-z0-9_-]*|'')
        echo "invalid customer id" >&2
        exit 1
        ;;
esac
```

Then construct:

```bash
LOCK="/run/jobs/customer-$CUSTOMER_ID.lock"
```

Coordination metadata is still filesystem input and must be handled safely.

---

## Lock ordering and deadlocks

If a job needs two locks:

```text
lock A
lock B
```

and another job acquires:

```text
lock B
lock A
```

they can deadlock.

Example:

```text
Process 1 holds A, waits for B
Process 2 holds B, waits for A
```

Neither can proceed.

Avoid multiple locks where possible.

If several locks are necessary, define a global order:

```text
always acquire customer lock before inventory lock
```

or:

```text
always sort lock names lexically before acquiring
```

Timeouts can prevent infinite waiting but do not solve the underlying ordering flaw.

---

## Lock hierarchy

A system may define:

```text
global maintenance lock
branch lock
customer lock
resource lock
```

A consistent hierarchy might be:

```text
global
then branch
then customer
then resource
```

Every process follows the same order.

This is standard concurrency engineering.

Cron jobs can participate in exactly the same deadlock patterns as multithreaded programs, only at process level.

---

## Do not hold a lock while waiting unnecessarily

Suppose:

```bash
acquire_lock
download_5GB_file
update_database
release_lock
```

The lock is held during a slow network transfer even though only the database update requires exclusivity.

This increases contention.

A better structure may be:

```bash
download_to_unique_temp_file

acquire_lock
validate_current_state
update_database
release_lock

delete_temp
```

But moving work outside the lock introduces consistency questions.

For example:

```text
Did state change while download occurred?
```

Sometimes revalidation is required after lock acquisition.

Concurrency design is about balancing correctness and lock duration.

---

## Optimistic concurrency

Not every workflow requires locking before work begins.

An optimistic model assumes conflicts are rare and detects them at commit time.

Example:

```text
read record version=10
calculate update
UPDATE ... WHERE version=10
```

If another process changed the row to version 11, the update affects zero rows and the program retries.

This is common in databases and APIs.

For cron jobs managing shared remote state, optimistic concurrency can be stronger than filesystem locks because it protects the actual resource.

---

## Atomic rename as a local publication primitive

A common safe pattern:

```bash
tmp=$(mktemp /var/lib/app/.report.XXXXXX)

generate_report > "$tmp"

mv "$tmp" /var/lib/app/report
```

If source and destination are on the same filesystem, rename is generally atomic from the perspective of path replacement.

Readers see either:

```text
old report
```

or:

```text
new report
```

not a half-written intermediate.

Use this for:

```text
JSON snapshots
configuration generation
reports
status files
indexes
```

when replacing a complete artifact.

Atomic rename does not replace locking when writer order matters, but it dramatically improves reader safety.

---

## Do not create the temporary file on another filesystem

Bad:

```bash
tmp=$(mktemp /tmp/report.XXXXXX)
generate > "$tmp"
mv "$tmp" /var/lib/app/report
```

If:

```text
/tmp
```

and:

```text
/var/lib/app
```

are different filesystems, `mv` cannot use a single rename operation.

It may copy and delete.

The final replacement is no longer the same atomic filesystem operation.

Create the temp file in the destination directory:

```bash
tmp=$(mktemp /var/lib/app/.report.XXXXXX)
```

Then:

```bash
mv "$tmp" /var/lib/app/report
```

is much more likely to use atomic rename semantics.

---

## `noclobber` is not a general lock

Shell option:

```bash
set -C
```

can prevent `>` from overwriting an existing regular file in some shells.

People sometimes use:

```bash
set -C
echo $$ > /run/job.lock
```

as a lock acquisition attempt.

This can improve atomic creation compared with test-then-touch patterns, but portability and symlink semantics require care.

For Linux cron jobs, `flock` is usually clearer and easier to audit.

Use a standard locking primitive rather than clever shell redirection when possible.

---

## Hard links as locks are possible but uncommon

POSIX-style lock algorithms can use:

```text
link()
mkdir()
O_CREAT|O_EXCL
```

because these provide atomic namespace operations.

They are useful when `flock` is unavailable or when portable filesystem locking is required.

But implementing crash recovery correctly is easy to get wrong.

For ordinary Linux administration:

```text
flock
```

is the practical default.

---

## Locking does not fix non-idempotent external effects

Suppose a job sends emails:

```bash
for customer in ...; do
    send_invoice_email
done
```

A lock prevents two copies from running simultaneously.

But if the process crashes after sending 500 of 1000 emails and is rerun, the first 500 may be sent again.

This is a restart/idempotency problem, not a concurrency problem.

Reliable jobs may need durable progress state:

```text
which items were completed
transactional outbox
message queue
unique operation ID
database status
```

Concurrency control alone does not guarantee exactly-once effects.

---

## "Exactly once" is harder than it sounds

For operations involving remote systems, a process can experience:

```text
request sent
remote system performs action
network response lost
local process cannot tell whether action happened
```

If it retries, the operation may happen twice.

Cron cannot solve this.

Reliable systems use:

```text
idempotency keys
unique transaction IDs
database constraints
deduplication tables
transactional queues
remote APIs with idempotent semantics
```

A scheduled task should avoid claiming:

```text
exactly once
```

unless the underlying architecture actually provides it.

---

## Idempotency keys

Suppose a daily billing job creates a payment request.

Use a key such as:

```text
billing:customer-42:2026-09-13
```

The remote API or local database treats repeated requests with the same key as one logical operation.

Now:

```text
retry
duplicate cron
process crash
```

do not automatically create duplicate charges.

This is much stronger than hoping the scheduler runs only once.

---

## Checkpointing long jobs

A cron job that processes 10 million records may take hours.

If it crashes at record 9 million and always restarts from the beginning, reliability is poor.

Checkpoint state:

```text
last processed ID
batch number
cursor
timestamp
partition
```

Example database table:

```text
job_name
checkpoint
updated_at
```

The job can resume safely.

Checkpoint updates should themselves be atomic and consistent with processed work.

Application-level job systems often provide this more naturally than shell scripts.

---

## Batch processing reduces lock duration

Instead of:

```text
lock entire 3-hour job
```

process in small transactional batches.

Conceptually:

```text
acquire batch ownership
process 1000 rows
commit
release
repeat
```

This allows controlled parallelism and easier retries.

At this point the problem has moved beyond basic cron and into job-processing architecture.

That is often the correct move.

Cron should trigger complex systems, not become one.

---

## `nice` does not provide concurrency control

This:

```bash
nice -n 10 /opt/jobs/report
```

changes CPU scheduling priority.

It does not prevent two reports from running.

Likewise:

```bash
ionice
```

changes I/O scheduling behavior where supported.

Neither is a lock.

A job may combine:

```bash
flock -n /run/report.lock \
    nice -n 10 \
    ionice -c2 -n7 \
    /opt/jobs/report
```

but each tool solves a different problem.

---

## CPU affinity does not prevent overlap

This:

```bash
taskset -c 2 /opt/jobs/task
```

pins or restricts CPU affinity.

Two copies can still run on CPU 2.

Resource placement and serialization are separate concerns.

---

## cgroups do not automatically serialize work

A systemd scope or cgroup can limit:

```text
CPU
memory
I/O
process count
```

but multiple scheduled instances can still coexist inside or across cgroups unless concurrency is explicitly limited.

Resource control and concurrency control should not be confused.

---

## `pgrep` is not a reliable lock

A common script:

```bash
if pgrep -f '/opt/jobs/report' >/dev/null; then
    exit 0
fi

/opt/jobs/report-real
```

This is fragile.

Problems:

```text
race after pgrep
pattern may match itself
pattern may match unrelated command
old process may exit immediately after test
new process may start immediately after test
```

This is another:

```text
check then act
```

race.

Use a real atomic lock primitive.

Process inspection is for diagnosis, not synchronization.

---

## `pidof` and `ps | grep` are not locks either

These patterns:

```bash
pidof job
```

and:

```bash
ps aux | grep job
```

are useful for observation.

They are not atomic coordination.

Never implement correctness around:

```text
I did not see another process
therefore it is safe to start
```

Between the check and process start, another instance can appear.

---

## File creation with `O_EXCL`

Programming languages can atomically create a lock file with:

```text
O_CREAT | O_EXCL
```

The open succeeds only if the pathname does not already exist.

Python example conceptually:

```python
os.open(
    path,
    os.O_CREAT | os.O_EXCL | os.O_WRONLY,
    0o600
)
```

This can be used to build lock mechanisms.

But stale lock recovery still needs design if the process dies.

Libraries that wrap OS locking primitives are preferable to custom lock protocols.

---

## Locking in Python

Python can use `fcntl.flock` on Unix-like systems.

Example:

```python
import fcntl
import os
import sys

fd = os.open(
    "/run/report.lock",
    os.O_CREAT | os.O_RDWR,
    0o600,
)

try:
    fcntl.flock(
        fd,
        fcntl.LOCK_EX | fcntl.LOCK_NB,
    )
except BlockingIOError:
    print("already running")
    sys.exit(0)

run_job()
```

The descriptor must remain open while the lock is required.

When the process exits, the lock is released.

This places concurrency logic inside the application instead of shell wrappers.

For production code, error handling and ownership need more care, but the model is clean.

---

## Locking in PHP

PHP supports file locking through:

```php
flock()
```

Conceptually:

```php
$fp = fopen('/run/app-report.lock', 'c');

if (!flock($fp, LOCK_EX | LOCK_NB)) {
    exit(0);
}

runReport();
```

Keep the file handle alive while the lock is needed.

Frameworks may provide higher-level mutex abstractions that work across Redis, databases, or caches.

Use the abstraction that matches deployment topology.

---

## Locking in Rust

Rust programs commonly rely on crates or OS APIs for file locking.

The important architecture is the same:

```text
open stable lock file
attempt exclusive non-blocking lock
if unavailable -> skip
hold descriptor while working
exit -> descriptor closes -> lock released
```

A native application can own its concurrency semantics directly rather than relying on cron syntax.

This often improves testability.

---

## Signals and locks

With kernel-backed file locks, a normal process exit releases the lock.

If the script traps termination:

```bash
trap cleanup TERM INT EXIT
```

do not manually remove the lock file while the process still holds an open descriptor unless you fully understand the implications.

Usually:

```text
leave pathname in place
let descriptor closure release lock
```

is safest.

Cleanup should focus on:

```text
temporary files
partial artifacts
application state
```

rather than deleting the lock pathname.

---

## A job killed with SIGKILL still releases `flock`

`SIGKILL` cannot be caught.

If the lock-holding process is terminated:

```bash
kill -9 PID
```

the kernel tears down its file descriptors.

The advisory lock disappears.

The lock file may remain.

Check:

```bash
lslocks | grep lockfile
```

before and after.

This behavior is one reason `flock` avoids stale lock-state problems.

---

## But child processes can inherit the lock descriptor

File descriptor inheritance can extend lock lifetime.

Suppose a shell acquires FD 9 and starts a child that inherits FD 9.

If the parent exits but the child remains alive with the descriptor open, the lock may remain held.

This can surprise administrators.

Inspect:

```bash
lsof /run/job.lock
```

and process trees:

```bash
pstree -ap PID
```

When using descriptor-style locking, understand whether spawned background processes inherit the lock descriptor.

One reason to avoid backgrounding work inside locked cron scripts is that lock lifetime becomes less obvious.

---

## Background processes are dangerous in cron wrappers

Bad:

```bash
#!/bin/sh

long_task &
exit 0
```

Cron sees the wrapper finish.

The child continues.

If the lock belongs only to the wrapper and is not inherited, a second run can begin while the first child remains active.

Even if the lock descriptor is inherited, process ownership becomes confusing.

Cron jobs should usually stay in the foreground.

Let the scheduled process represent the lifetime of the work.

If daemonization is required, use a service manager.

---

## `nohup` is usually the wrong tool in cron

A common pattern:

```cron
* * * * * nohup /opt/jobs/task &
```

is usually unnecessary.

Cron already runs without a terminal.

Backgrounding breaks lifecycle tracking.

`nohup` changes signal/output behavior but does not make cron more reliable.

A scheduled job should normally run in the foreground and finish when the work is complete.

For persistent services, use systemd or another supervisor.

---

## Overlap can amplify failures

Suppose a failing API call waits 60 seconds.

Cron launches every minute.

Normally one process exists.

The remote service becomes slow and each call takes 15 minutes.

Now approximately fifteen processes accumulate.

Each creates more remote requests, making the service even slower.

This positive feedback loop is called a retry or overload storm in broader distributed-systems contexts.

Non-overlap control can protect both local and remote systems.

---

## Concurrency limits beyond one

Sometimes the desired policy is not:

```text
maximum 1
```

but:

```text
maximum 4
```

`flock` provides mutual exclusion, not a counting semaphore.

For controlled parallelism, use:

```text
job queue
worker pool
GNU parallel with limits
application worker system
systemd service design
container orchestrator
```

Cron can enqueue work once.

Workers can enforce concurrency.

Do not implement a fragile semaphore from several lock files unless the environment is simple and well-tested.

---

## Queue-based architecture

Cron:

```cron
* * * * * /opt/app/enqueue-due-work
```

This job performs only:

```text
discover due tasks
enqueue them
```

Worker processes consume the queue with configured concurrency:

```text
4 workers
```

Benefits:

```text
retry
backpressure
visibility
concurrency limits
durable jobs
per-task status
```

This architecture is much stronger for large workloads than having cron directly perform every operation.

---

## Backpressure

Backpressure means the system has a way to slow producers when consumers cannot keep up.

Traditional cron has no built-in backpressure.

If a task is scheduled every minute, cron continues launching every minute.

A queue can represent backlog explicitly.

Workers can process at a sustainable rate.

Monitoring can alert:

```text
queue depth increasing
oldest job age too high
```

For workloads whose execution time varies widely, queues are often superior to direct cron execution.

---

## Lock contention should be observable

A job that silently skips because of a lock can hide chronic overload.

Log:

```text
event=skip reason=lock_held
```

Count it.

Example:

```bash
#!/usr/bin/env bash
set -u

exec 9>/run/sync.lock

if ! flock -n 9; then
    logger -t sync-job \
        "event=skip reason=lock_held"
    exit 0
fi

...
```

If monitoring sees:

```text
60 skips in one hour
```

the schedule or runtime has a problem.

A lock should prevent damage.

It should not conceal capacity issues.

---

## A long-running-job monitor

Store:

```text
start timestamp
PID
run ID
```

for observation, not locking.

Example:

```bash
STATE=/run/company-report.state

printf 'pid=%s\nstarted=%s\n' \
    "$$" \
    "$(date -Is)" \
    > "$STATE"
```

The real lock still uses `flock`.

Monitoring can inspect the state file to report:

```text
job running for 72 minutes
```

Do not use the state file itself as the synchronization primitive.

Separate:

```text
coordination
observability
```

---

## A robust lock wrapper

Example:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

JOB=inventory-sync
LOCK=/run/inventory-sync.lock
START=$(date +%s)
RUN_ID="$(date -u +%Y%m%dT%H%M%SZ)-$$"

exec 9>"$LOCK"

if ! flock -n 9; then
    logger -t "$JOB" \
        "run_id=$RUN_ID event=skip reason=lock_held"
    exit 0
fi

logger -t "$JOB" \
    "run_id=$RUN_ID event=start"

finish() {
    rc=$?
    end=$(date +%s)

    if (( rc == 0 )); then
        event=finish
        priority=user.info
    else
        event=fail
        priority=user.err
    fi

    logger -p "$priority" -t "$JOB" \
        "run_id=$RUN_ID event=$event rc=$rc duration_seconds=$((end-START))"

    exit "$rc"
}

trap finish EXIT

exec_job() {
    /opt/inventory/bin/sync
}

exec_job
```

Cron:

```cron
*/5 * * * * /opt/inventory/bin/run-sync
```

This provides:

```text
non-overlap
skip visibility
run ID
duration
exit status
kernel-managed lock release
```

---

## Add a timeout to the locked job

A job can acquire a lock and hang forever.

Then every future run skips forever.

Combine with a timeout:

```bash
/usr/bin/timeout \
    --kill-after=30s \
    20m \
    /opt/inventory/bin/sync
```

Inside the locked wrapper:

```bash
if /usr/bin/timeout \
    --kill-after=30s \
    20m \
    /opt/inventory/bin/sync
then
    rc=0
else
    rc=$?
fi
```

Now the lock holder eventually exits.

This is useful for network jobs and external commands.

Choose timeout based on expected worst-case behavior, not arbitrary convenience.

---

## Timeout shorter than schedule interval

Cron:

```cron
*/10 * * * * ...
```

A reasonable outer timeout might be:

```text
9 minutes
```

if every new interval should get a chance to run.

But if legitimate work sometimes takes 15 minutes, such a timeout destroys valid runs.

Possible policies:

```text
increase schedule interval
skip overlapping invocations
allow longer job with lock
split task into batches
use queue
```

Timeout is not a substitute for capacity planning.

---

## Locking and retries

Suppose the job retries transient failures for ten minutes.

Cron launches every five minutes.

If lock is held across retries:

```text
second run skips
```

This is usually desirable.

If the lock is released between retry attempts, another invocation may enter and duplicate the work.

Define the logical run boundary.

The lock should usually cover:

```text
initial attempt
all internal retries
final success/failure
```

when those attempts belong to one operation.

---

## Exponential backoff

For transient remote failures:

```text
attempt 1 -> wait 5s
attempt 2 -> wait 10s
attempt 3 -> wait 20s
attempt 4 -> wait 40s
```

This reduces pressure on a failing dependency.

Example shell:

```bash
#!/bin/sh
set -u

delay=5
attempt=1
max=5

while [ "$attempt" -le "$max" ]; do
    if /opt/jobs/remote-operation; then
        exit 0
    fi

    if [ "$attempt" -eq "$max" ]; then
        break
    fi

    sleep "$delay"
    delay=$((delay * 2))
    attempt=$((attempt + 1))
done

exit 1
```

Hold the lock if retries are part of one logical execution.

Add random jitter in large fleets so many hosts do not retry at identical times.

---

## Jitter reduces synchronized load

If 500 servers run:

```cron
0 * * * * ...
```

all 500 may contact the same API at the top of the hour.

One approach is random delay:

```bash
sleep "$(( RANDOM % 300 ))"
```

in Bash.

But random delay complicates lock and runtime calculation.

Some schedulers offer native randomized delay.

On large fleets, centralized scheduling or systemd randomized timer delay may be cleaner.

Jitter is for load distribution.

It is not a concurrency lock.

---

## Lock first or jitter first?

Suppose all invocations share one local lock.

If the script:

```text
sleep random
then acquire lock
```

multiple copies can accumulate sleeping before lock acquisition.

If it:

```text
acquire lock
sleep random
```

the lock is held while no useful work happens.

The correct answer depends on the goal.

For one host, random jitter is often unnecessary.

For a fleet, per-host locks do not coordinate the fleet anyway.

This illustrates why unrelated mechanisms should not be combined casually.

---

## Idempotent cleanup jobs often need no lock

Example:

```bash
find /var/cache/app \
    -type f \
    -mtime +7 \
    -delete
```

If two copies run, they may both attempt to delete the same files.

Usually:

```text
one succeeds
other sees file disappear
```

Depending on `find` behavior, harmless warnings may occur.

If this is acceptable, a lock may not be necessary.

Do not add locks mechanically.

Locks add:

```text
contention
failure modes
operational complexity
```

Use them when shared-state correctness requires them.

---

## Package managers already have their own locks

Commands such as:

```text
apt
dpkg
dnf
rpm
```

use internal locking to protect package databases.

A cron wrapper should still avoid launching competing package operations unnecessarily.

If the package manager says:

```text
Could not get lock
```

do not delete its lock file blindly.

The lock may represent an active transaction.

Investigate:

```bash
ps
lsof
systemctl
```

Deleting lock files is not a general solution to lock contention.

---

## Never delete a lock merely because a job looks stuck

With `flock`, the visible file can persist forever.

Removing it while another process holds the original inode can create a second independent lock object.

That can allow concurrent execution.

Use:

```bash
lslocks
lsof
ps
```

to determine whether an active holder exists.

For application-specific PID or lease locks, follow that application's recovery procedure.

Lock cleanup requires understanding the locking mechanism.

---

## Lock recovery after reboot

Kernel `flock` locks disappear when processes disappear.

After reboot:

```text
no process
no kernel lock
```

If `/run` is tmpfs, the lock file itself may disappear too.

This makes recovery straightforward.

If the path is on persistent storage, the empty lock file can remain but does not block acquisition.

That is normal.

---

## Filesystem snapshots and locks

A filesystem snapshot captures files.

It does not necessarily capture active kernel lock state in a way that can later be restored as a live process lock.

After restoring a VM or filesystem snapshot, application-level state may indicate a job was running while no actual process exists.

This is another reason:

```text
persistent "running" flags
```

should not be used as the sole lock mechanism without recovery semantics.

---

## Application state flags need transactions

Suppose a table contains:

```text
job_running = true
```

The job sets it true at start and false at finish.

Crash:

```text
true remains forever
```

Now all future runs skip.

This is the database equivalent of a stale lock file.

If using database state for coordination, use:

```text
transactional advisory locks
leases with ownership
compare-and-swap
row locks
```

rather than a simple Boolean when crash recovery matters.

---

## Heartbeats for long-running distributed jobs

A distributed worker may store:

```text
owner_id
lease_until
last_heartbeat
```

and renew periodically.

Another worker can take over after the lease expires.

This supports crash recovery.

But it introduces lease-expiry races and clock/time assumptions.

For ordinary single-host cron, `flock` is much simpler.

Distributed coordination should be used only when the deployment truly requires it.

---

## Clock changes do not affect kernel file-lock lifetime

A local `flock` lock is tied to process/file-descriptor lifetime, not wall-clock timestamps.

Changing timezone or NTP clock does not expire it.

This is useful because local serialization remains stable across clock adjustments.

Distributed lease systems based on timestamps have different time semantics.

---

## Forking processes and lock ownership

A process that forks can pass open file descriptors to children.

The exact semantics of `flock` locks across `fork()` and descriptor duplication matter when building custom daemons.

For normal cron shell wrappers, the practical rule is:

```text
keep the lock descriptor open in the foreground process that represents the job
do not daemonize unexpectedly
```

If a program forks into the background, test lock ownership explicitly.

---

## `exec` simplifies lock lifetime

Script:

```bash
exec 9>/run/job.lock
flock -n 9 || exit 0

exec /opt/jobs/job-real
```

`exec` replaces the shell process with:

```text
job-real
```

while open descriptors can remain inherited unless marked close-on-exec.

This can make the real job process directly own the lock lifetime.

But descriptor behavior should be tested for the shell and tool usage.

A simpler wrapper that remains alive and waits for the child is also perfectly acceptable.

---

## Lock only after environment validation?

Suppose the script checks:

```text
configuration exists
destination mounted
binary installed
```

before acquiring a lock.

This reduces lock hold time.

But two processes can perform the same validation concurrently, which is normally harmless.

Then lock before modifying shared state.

A good layout:

```bash
validate_static_dependencies
prepare_unique_temp_data

acquire_lock

revalidate_shared_state
commit_update
```

The revalidation after acquiring the lock matters because shared state may change during preparation.

---

## Two-phase publication

For expensive generation:

```text
generate candidate independently
lock
compare current state
publish candidate
unlock
```

Example:

```bash
tmp=$(mktemp /var/lib/report/.candidate.XXXXXX)

generate_report > "$tmp"

exec 9>/run/report-publish.lock
flock 9

validate_candidate "$tmp"
mv "$tmp" /var/lib/report/current
```

Now report generation can run concurrently while publication is serialized.

Whether this is logically correct depends on data freshness requirements.

An older candidate could still publish after a newer one unless versions are compared.

---

## Version-aware publication

Include a generation version:

```text
source timestamp
database revision
sequence number
```

Before publication, compare it with current version.

Only publish if:

```text
candidate_version >= current_version
```

This prevents a slower old run from overwriting a newer result.

This is an example of solving ordering at the data layer rather than relying solely on process timing.

---

## Monotonic sequence numbers

If a database supplies a monotonically increasing revision:

```text
1042
1043
1044
```

store it alongside generated output.

A publisher can reject stale candidate:

```text
candidate=1042
current=1043
=> do not replace
```

This is useful when parallel generation is intentionally allowed.

---

## Cron frequency should reflect business semantics

Suppose data synchronization has a requirement:

```text
state should be no more than 10 minutes stale
```

Scheduling every minute may seem safer.

But if every run takes eight minutes and uses a lock, most invocations skip.

A more coherent schedule could be:

```cron
*/10 * * * * ...
```

with a timeout and monitoring.

Or use a continuously running worker if near-real-time state is actually required.

Cron is excellent for periodic work.

It is not ideal for simulating a daemon through extremely frequent schedules.

---

## One-minute cron is not a queue

This pattern:

```cron
* * * * * /opt/app/process-one-item
```

processes at most one item per minute.

If 1000 items arrive:

```text
1000 minutes minimum
```

People sometimes add more cron entries to increase throughput.

That quickly becomes hard to coordinate.

Use a worker loop or queue system.

Cron should trigger coarse-grained periodic activity, not emulate a concurrency scheduler.

---

## Cron and `systemd-run`

On systemd systems, a cron job can launch work through:

```bash
systemd-run
```

with resource controls or transient units.

This can provide stronger lifecycle tracking.

However, using cron to start systemd units may be unnecessary if a native systemd timer can schedule the unit directly.

Do not stack schedulers without a reason.

---

## systemd service overlap behavior

A systemd timer activating a service behaves differently from cron in several ways depending on unit type and state.

If the same service unit is already active, another timer activation does not necessarily create another independent service instance.

This can naturally prevent some overlap.

Systemd also supports:

```text
Persistent=
RandomizedDelaySec=
AccuracySec=
RuntimeMaxSec=
```

and service-level hardening.

Cron remains simpler and portable.

The point is not that one is universally better, but that concurrency semantics differ.

---

## Kubernetes CronJob concurrency policies

In container orchestration, Kubernetes CronJob resources support policies conceptually like:

```text
Allow
Forbid
Replace
```

for concurrent executions.

This is a direct example of a scheduler exposing concurrency policy natively.

Traditional cron does not.

If workloads already run inside Kubernetes, use the orchestrator's scheduling and concurrency primitives rather than embedding host cron without reason.

---

## `Replace` semantics are dangerous for stateful jobs

A policy like:

```text
start new run and terminate old run
```

can be appropriate for refresh operations.

It can be disastrous for:

```text
database backup
schema migration
filesystem transaction
billing
```

because termination can leave partial state.

Any "replace old job" design requires application-level cancellation safety.

Cron plus:

```bash
pkill old-job
start new-job
```

is rarely a good generic solution.

---

## Avoid killing by command-name pattern

Bad:

```bash
pkill -f '/opt/jobs/report'
```

This can match:

```text
current run
unrelated debugging process
another user's process
wrapper shell
```

If controlled cancellation is required, use a service manager or application-specific process identity.

Synchronization by broad process-name matching is fragile.

---

## Graceful termination

A long-running job should respond to:

```text
SIGTERM
SIGINT
```

when practical.

Example Bash:

```bash
#!/usr/bin/env bash
set -euo pipefail

tmp=""

cleanup() {
    if [[ -n "${tmp:-}" ]]; then
        rm -f -- "$tmp"
    fi
}

trap cleanup EXIT TERM INT

tmp="$(mktemp /var/lib/app/.work.XXXXXX)"

do_work "$tmp"

mv "$tmp" /var/lib/app/result
tmp=""
```

If terminated before publication, the temporary file is removed.

Designing interruption safety is part of reliable scheduling.

---

## Do not update "success" state before releasing all critical work

Bad:

```bash
touch /var/lib/job/last-success
upload_remote
```

If upload fails, monitoring sees fresh success.

Good:

```bash
perform_local_work
upload_remote
verify_remote
touch /var/lib/job/last-success
```

If locking is used:

```text
lock
perform critical operation
record durable success
unlock
```

The ordering of state transitions matters.

---

## Transactions can replace locks for some database jobs

Suppose two cron workers update inventory.

A database transaction with row-level locking:

```text
BEGIN
SELECT ... FOR UPDATE
modify
COMMIT
```

can protect shared records.

This is stronger than host-local `flock` in multi-server deployments.

Locks should be located near the resource they protect.

Filesystem lock for filesystem resource.

Database lock for shared database state.

Distributed coordination for distributed resource.

---

## Avoid giant global locks

A global lock:

```text
/run/all-maintenance.lock
```

around:

```text
backup
cleanup
report
sync
indexing
```

prevents all maintenance jobs from overlapping.

This is simple but can create unnecessary delays.

Instead, identify actual conflicts.

Maybe:

```text
backup and cleanup conflict
report is independent
sync is independent
```

Then use targeted locks.

Concurrency design should reflect resource relationships.

---

## A conflict matrix

For complex maintenance, write down conflicts.

Example:

```text
             backup  cleanup  report  reindex
backup         X       X       -       X
cleanup        X       X       -       -
report         -       -       X       -
reindex        X       -       -       X
```

This makes coordination requirements explicit.

If the matrix becomes complicated, a maintenance orchestrator may be better than independent cron entries.

---

## Staggering schedules is not locking

Administrators sometimes avoid overlap by scheduling:

```cron
0 2 * * * backup
0 3 * * * cleanup
0 4 * * * report
```

This works only if runtime assumptions remain true.

If backup runs for two hours:

```text
cleanup starts during backup
```

Staggering reduces expected collision.

It does not enforce exclusion.

If overlap would be unsafe, use an actual coordination mechanism.

---

## Random delays are not locking either

Similarly:

```bash
sleep "$(( RANDOM % 600 ))"
```

can reduce simultaneous starts.

Two jobs can still overlap.

Use jitter for load shaping.

Use locks for correctness.

---

## "No overlap" does not mean "healthy"

Suppose:

```cron
* * * * * flock -n /run/job.lock /opt/jobs/job
```

The job hangs for three days.

No duplicate copies run.

The lock successfully prevents overlap.

The system is still broken.

Monitor:

```text
last start
last finish
duration
last success
skip count
```

Locking is one reliability control among several.

---

## Detect a permanently held lock

If every run logs:

```text
event=skip reason=lock_held
```

inspect:

```bash
lslocks | grep job.lock
```

then:

```bash
ps -o pid,ppid,user,stat,etime,cmd -p PID
```

Check:

```bash
lsof -p PID
```

and:

```bash
strace -p PID
```

if appropriate.

Do not remove the file first.

Find the holder.

---

## A stale manual directory lock

If using:

```bash
mkdir /run/job.lockdir
```

and the process crashed, the directory remains.

A recovery approach may store metadata:

```text
PID
hostname
boot ID
start time
```

inside:

```text
/run/job.lockdir/owner
```

Then administrators can determine whether the owner still exists.

But automated stale-lock removal is difficult because PID reuse and cross-host states can mislead.

Again, local Linux jobs should generally prefer `flock`.

---

## Boot ID helps identify stale metadata

Linux exposes:

```bash
cat /proc/sys/kernel/random/boot_id
```

This UUID changes on boot.

If a persistent state file says:

```text
boot_id=OLD
pid=1234
```

and current boot ID differs, the recorded process cannot still exist from the previous boot.

This is useful for application-level recovery metadata.

It is usually unnecessary with kernel file locks.

---

## A complete single-host cron pattern

Script:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

JOB=customer-sync
LOCK=/run/customer-sync.lock
STATE=/var/lib/customer-sync
RUN_ID="$(date -u +%Y%m%dT%H%M%SZ)-$$"
START="$(date +%s)"

mkdir -p "$STATE"

exec 9>"$LOCK"

if ! flock -n 9; then
    logger -t "$JOB" \
        "run_id=$RUN_ID event=skip reason=lock_held"
    exit 0
fi

finish() {
    rc=$?
    end=$(date +%s)

    if (( rc == 0 )); then
        event=finish
        priority=user.info
    else
        event=fail
        priority=user.err
    fi

    logger -p "$priority" -t "$JOB" \
        "run_id=$RUN_ID event=$event rc=$rc duration_seconds=$((end-START))"

    exit "$rc"
}

trap finish EXIT

logger -t "$JOB" \
    "run_id=$RUN_ID event=start"

date -Is > "$STATE/last-start.tmp"
mv "$STATE/last-start.tmp" "$STATE/last-start"

/usr/bin/timeout \
    --kill-after=30s \
    20m \
    /opt/customer-sync/bin/sync

date -Is > "$STATE/last-success.tmp"
mv "$STATE/last-success.tmp" "$STATE/last-success"
```

Cron:

```cron
*/15 * * * * /opt/customer-sync/bin/run-sync
```

This design has:

```text
local non-overlap
skip visibility
runtime limit
start/success state
atomic state-file replacement
exit logging
duration logging
```

It still assumes one host.

For multiple hosts, coordination must move to a shared system.

---

## Lab: observe cron overlap

Create:

```bash
cat > /tmp/cron-overlap.sh <<'EOF'
#!/bin/sh

echo "start pid=$$ time=$(date -Is)" \
    >> /tmp/cron-overlap.log

sleep 90

echo "finish pid=$$ time=$(date -Is)" \
    >> /tmp/cron-overlap.log
EOF

chmod +x /tmp/cron-overlap.sh
```

Schedule temporarily:

```cron
* * * * * /tmp/cron-overlap.sh
```

After several minutes:

```bash
cat /tmp/cron-overlap.log
```

You should see overlapping start times.

Inspect:

```bash
pgrep -af cron-overlap
```

This proves cron does not wait automatically.

Remove the temporary cron entry after the lab.

---

## Lab: prevent overlap with `flock`

Modify:

```bash
cat > /tmp/cron-overlap.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

exec 9>/tmp/cron-overlap.lock

if ! flock -n 9; then
    echo "skip pid=$$ time=$(date -Is)" \
        >> /tmp/cron-overlap.log
    exit 0
fi

echo "start pid=$$ time=$(date -Is)" \
    >> /tmp/cron-overlap.log

sleep 90

echo "finish pid=$$ time=$(date -Is)" \
    >> /tmp/cron-overlap.log
EOF
```

Keep:

```cron
* * * * * /tmp/cron-overlap.sh
```

Now the log should show:

```text
start
skip
finish
start
skip
finish
```

rather than multiple active workers.

This is the most direct demonstration of cron locking.

---

## Lab: prove lock release after forced termination

Run:

```bash
/tmp/cron-overlap.sh &
```

Find PID:

```bash
pgrep -af cron-overlap
```

Check:

```bash
lslocks | grep cron-overlap.lock
```

Kill:

```bash
kill -9 PID
```

Then:

```bash
lslocks | grep cron-overlap.lock
```

The lock should no longer be active even though:

```bash
ls -l /tmp/cron-overlap.lock
```

still shows the file.

Run the script again.

It should acquire the lock.

This demonstrates the difference between:

```text
lock file pathname
kernel lock state
```

---

## Lab: show why a manual lock file becomes stale

Create:

```bash
cat > /tmp/bad-lock.sh <<'EOF'
#!/bin/sh

LOCK=/tmp/bad-lock.file

if [ -e "$LOCK" ]; then
    echo "locked"
    exit 0
fi

touch "$LOCK"

echo "working pid=$$"
sleep 60

rm -f "$LOCK"
EOF

chmod +x /tmp/bad-lock.sh
```

Run:

```bash
/tmp/bad-lock.sh &
```

Then:

```bash
kill -9 PID
```

Check:

```bash
ls -l /tmp/bad-lock.file
```

The file remains.

Future runs print:

```text
locked
```

forever.

This is the stale-lock problem.

---

## Lab: show the test-and-create race

A deterministic race can be easier to demonstrate by adding a sleep.

Create:

```bash
cat > /tmp/racy-lock.sh <<'EOF'
#!/bin/sh

LOCK=/tmp/racy-lock

if [ ! -e "$LOCK" ]; then
    echo "$$ saw no lock"
    sleep 2
    touch "$LOCK"
    echo "$$ entered critical section"
    sleep 5
    rm -f "$LOCK"
else
    echo "$$ saw lock"
fi
EOF

chmod +x /tmp/racy-lock.sh
```

Run two copies almost simultaneously:

```bash
/tmp/racy-lock.sh &
/tmp/racy-lock.sh &
wait
```

Possible output:

```text
6100 saw no lock
6101 saw no lock
6100 entered critical section
6101 entered critical section
```

Both passed the check.

This makes the race visible.

---

## Lab: atomic `mkdir` acquisition

Create:

```bash
cat > /tmp/mkdir-lock.sh <<'EOF'
#!/bin/sh

LOCK=/tmp/mkdir-lock.dir

if mkdir "$LOCK" 2>/dev/null; then
    echo "$$ acquired"
    sleep 5
    rmdir "$LOCK"
else
    echo "$$ blocked"
fi
EOF

chmod +x /tmp/mkdir-lock.sh
```

Run:

```bash
/tmp/mkdir-lock.sh &
/tmp/mkdir-lock.sh &
wait
```

Only one should acquire.

Then kill the owner before cleanup and observe that the directory remains.

This demonstrates:

```text
atomic acquisition
but stale state
```

---

## Lab: atomic publication

Create a reader:

```bash
cat > /tmp/read-report.sh <<'EOF'
#!/bin/sh

while true; do
    if [ -f /tmp/report-final ]; then
        wc -c /tmp/report-final
    fi
    sleep 0.1
done
EOF

chmod +x /tmp/read-report.sh
```

A weak writer:

```bash
cat > /tmp/write-report-bad.sh <<'EOF'
#!/bin/sh

: > /tmp/report-final

i=0
while [ "$i" -lt 1000 ]; do
    printf 'line %s\n' "$i" \
        >> /tmp/report-final
    i=$((i + 1))
done
EOF
```

Readers can observe intermediate sizes.

Now strong writer:

```bash
cat > /tmp/write-report-good.sh <<'EOF'
#!/bin/sh
set -eu

tmp=$(mktemp /tmp/.report.XXXXXX)
trap 'rm -f "$tmp"' EXIT

i=0
while [ "$i" -lt 1000 ]; do
    printf 'line %s\n' "$i" \
        >> "$tmp"
    i=$((i + 1))
done

mv "$tmp" /tmp/report-final
trap - EXIT
EOF
```

Readers see complete old or new file much more cleanly.

This demonstrates atomic publication.

---

## Lab: database uniqueness as duplicate protection

Using a test database, define a table conceptually:

```sql
CREATE TABLE daily_job (
    run_date date PRIMARY KEY,
    generated_at timestamptz NOT NULL
);
```

Two processes attempt:

```sql
INSERT INTO daily_job(run_date, generated_at)
VALUES (CURRENT_DATE, now());
```

One succeeds.

The other receives a uniqueness violation.

Now change to an upsert appropriate for the database.

The schema itself prevents duplicate logical records even if scheduler duplication occurs.

This is a stronger invariant than relying only on cron.

---

## Production example: backup locking

A backup script should often reject concurrent execution.

Example:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

LOCK=/run/app-backup.lock
BACKUP_DIR=/var/backups/app

exec 9>"$LOCK"

if ! flock -n 9; then
    logger -t app-backup \
        "event=skip reason=already_running"
    exit 0
fi

tmp="$(mktemp "$BACKUP_DIR/.backup.XXXXXX.tar.gz")"

cleanup() {
    rm -f -- "$tmp"
}

trap cleanup EXIT

tar -czf "$tmp" \
    -C /srv/app \
    data

gzip -t "$tmp"

final="$BACKUP_DIR/app-$(date -u +%Y%m%dT%H%M%SZ).tar.gz"

mv "$tmp" "$final"
trap - EXIT

logger -t app-backup \
    "event=finish file=$final"
```

Using a unique timestamped final filename avoids overwriting another backup.

The lock prevents concurrent backups competing for:

```text
disk I/O
application snapshot state
remote upload
```

---

## Production example: rsync mirror

Cron:

```cron
*/10 * * * * /opt/jobs/mirror
```

Script:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

exec 9>/run/site-mirror.lock

if ! flock -n 9; then
    logger -t site-mirror \
        "event=skip reason=already_running"
    exit 0
fi

/usr/bin/rsync \
    -a \
    --delete \
    /srv/source/ \
    /srv/mirror/
```

`--delete` makes overlap especially undesirable because two processes can make decisions against changing directory state.

Serialization simplifies correctness.

---

## Production example: report generation without unnecessary lock duration

Report generation:

```text
database query -> 8 minutes
render PDF -> 2 minutes
publish current.pdf -> milliseconds
```

If queries are safe concurrently, do not necessarily lock all ten minutes.

Pattern:

```bash
tmp=$(mktemp /var/lib/report/.report.XXXXXX.pdf)

generate_report > "$tmp"

exec 9>/run/report-publish.lock
flock 9

mv "$tmp" /var/lib/report/current.pdf
```

But if runs use different snapshots of data, a slow older run may still overwrite a newer one.

Add a version check or serialize the entire generation if freshness ordering matters.

This example shows why locking scope is a data-consistency decision.

---

## Production example: email digest

A digest job sends one email per day.

Weak:

```cron
0 8 * * * /opt/jobs/send-digest
```

If cron runs twice, two emails are sent.

A lock prevents simultaneous duplicates but not sequential reruns.

Use durable idempotency:

```text
digest_date unique key
```

Database:

```sql
daily_digest(date PRIMARY KEY, sent_at ...)
```

Workflow:

```text
begin transaction
claim date if not already sent
generate message
send with idempotency key if mail API supports it
record completion
```

The exact transaction boundary depends on email provider semantics.

The important point is:

```text
one-per-day business guarantee
```

must be represented in durable state.

Cron scheduling alone cannot guarantee it.

---

## Production example: cleanup

Cleanup:

```bash
find /var/cache/app \
    -type f \
    -mtime +7 \
    -delete
```

If the job is idempotent and duplicate execution is harmless, locking may be unnecessary.

This is a good contrast.

Reliable engineering does not mean:

```text
add flock everywhere
```

It means:

```text
understand shared state
choose only the controls needed
```

---

## Production example: certificate renewal

Many certificate-renewal tools already implement their own concurrency and state checks.

Wrapping them in another custom lock may be redundant.

Before adding external locking, understand the application's documented behavior.

Layering unknown locks can create:

```text
deadlocks
unnecessary skips
confusing failure codes
```

Use native concurrency controls when they are well-designed.

---

## Production example: database maintenance

Suppose:

```cron
0 3 * * * postgres /opt/jobs/db-maint
```

The database itself may already serialize:

```text
VACUUM
schema changes
index maintenance
```

through database locks.

That does not mean multiple maintenance jobs should be launched carelessly.

Application-level lock:

```text
one maintenance workflow at a time
```

can improve predictability.

Database-level locks protect internal consistency.

Both layers can have value.

---

## Cron plus maintenance window

A system may define:

```text
maintenance window 02:00-04:00
```

Jobs:

```text
02:00 backup
02:30 cleanup
03:00 reindex
```

Locks can enforce unsafe combinations.

But if backup regularly runs past 04:00, the maintenance window itself is violated.

Monitor:

```text
duration
finish time
overrun
```

Concurrency controls should support capacity planning, not hide overruns.

---

## Resource exhaustion from many waiting lock processes

Blocking `flock` consumes relatively little CPU, but thousands of waiting processes still consume:

```text
PID space
memory
process table entries
file descriptors
operational visibility
```

A misconfigured per-minute cron can build a large queue over days.

Use:

```bash
pgrep -af flock
```

and:

```bash
ps -eo stat,cmd | grep ...
```

If waiting is not semantically required, use non-blocking mode.

---

## Lock starvation

When many processes wait for a lock, assumptions about exact fairness should be avoided unless documented.

Do not design business correctness around:

```text
the oldest waiter definitely acquires next
```

If strict ordering matters, use a queue with explicit ordering.

Traditional advisory locks provide exclusion, not a full scheduling policy.

---

## Fairness versus freshness

A report job might prefer freshness:

```text
skip old queued runs
run newest state
```

A financial event processor might prefer fairness/order:

```text
process every event in sequence
```

Those are different workload classes.

Cron plus non-blocking lock fits freshness-oriented reconciliation jobs.

A durable ordered queue fits event-processing jobs.

---

## Reconciliation jobs are naturally cron-friendly

A reconciliation job asks:

```text
What should current state be?
Make actual state match desired state.
```

Examples:

```text
sync DNS records
rebuild cache
recalculate dashboard
remove expired files
refresh inventory snapshot
```

If one interval is skipped, the next run can usually reconcile everything.

This makes:

```text
flock -n
```

a natural concurrency policy.

---

## Event jobs are less cron-friendly

An event job says:

```text
perform one action for each event
```

Examples:

```text
charge invoice
send order email
process webhook
record trade
```

Skipping a run can lose work if events are represented only by schedule time.

Use durable event storage and queues.

Cron can periodically scan for unprocessed events, but the source of truth must be durable.

---

## Use state transitions, not schedule assumptions

Instead of:

```text
at 02:00 process yesterday's rows
```

prefer:

```text
find rows where status = pending
claim safely
process
mark complete
```

Now if 02:00 execution is missed, 02:10 or tomorrow's execution can still process pending work.

This is resilient to:

```text
machine downtime
cron failure
lock skip
temporary dependency failure
manual retry
```

Cron becomes a trigger, not the source of truth.

---

## Claiming work atomically

For database-backed jobs, workers can atomically claim rows.

Patterns include:

```text
UPDATE ... WHERE status='pending' ... RETURNING
SELECT ... FOR UPDATE SKIP LOCKED
lease columns
queue tables
```

Database-specific syntax varies.

This allows multiple workers to process different items safely.

It is a more scalable concurrency model than a single global file lock.

---

## `SKIP LOCKED` style processing

Some relational databases support row locking with skip semantics.

Conceptually:

```text
worker A locks rows 1-100
worker B skips those and locks 101-200
```

This gives parallel processing without duplicate ownership.

At that point cron might simply ensure workers are started.

The database manages item-level coordination.

---

## Cron should not own business transaction integrity

A schedule:

```cron
0 0 * * * billing
```

does not guarantee:

```text
billing happens exactly once
billing happens only once
billing fully completes
```

Business integrity should be enforced through:

```text
database constraints
transactions
idempotency
durable state
```

Cron only answers:

```text
when should we attempt the workflow?
```

This distinction is one of the most important lessons in reliable scheduler design.

---

## A concurrency troubleshooting workflow

If a cron job produces duplicate effects, start by finding duplicate processes:

```bash
pgrep -af job-name
```

Check process tree:

```bash
pstree -ap
```

Search every schedule source:

```bash
crontab -l
sudo crontab -l
sudo grep -R -n 'job-name' /etc/cron* 2>/dev/null
systemctl list-timers --all | grep -i job
```

Check multiple hosts:

```text
same app deployed elsewhere?
container replicas?
old server still active?
DR node?
```

Inspect logs for:

```text
same timestamp
different PID
different host
different run ID
```

Then determine whether duplication comes from:

```text
overlap on one host
duplicate scheduler definitions
multiple hosts
application retry
manual execution
```

Each cause requires a different fix.

---

## Diagnosing a lock that seems ineffective

Suppose two copies run despite:

```bash
flock -n /run/job.lock ...
```

Check:

```text
Are both commands using the same exact path?
Are they in different containers?
Are they in different mount namespaces?
Are they on different hosts?
Was the lock file deleted and recreated?
Is one execution bypassing the wrapper?
Does the application fork and release the lock unexpectedly?
```

Inspect:

```bash
readlink /proc/PID/ns/mnt
```

for mount namespaces.

Compare:

```bash
stat /run/job.lock
```

inside each environment.

Two containers can both have:

```text
/run/job.lock
```

but those paths may refer to different filesystems.

String equality is not coordination equality.

---

## Container lock visibility

Container A:

```text
/run/job.lock
```

Container B:

```text
/run/job.lock
```

If each has a private `/run`, each gets its own lock.

If a shared volume is mounted at:

```text
/locks
```

then:

```text
/locks/job.lock
```

may represent a shared file.

But network/distributed filesystem lock semantics must still be considered.

For container replicas, orchestrator-native concurrency or database coordination is often preferable.

---

## Namespaces can isolate process visibility too

Inside a container:

```bash
pgrep -af job
```

may not see a duplicate process running in another PID namespace.

Therefore process-name checks become even less reliable in containerized systems.

Centralized coordination should not depend on local process visibility when replicas exist.

---

## Monitoring lock wait time

For blocking locks, measure how long acquisition takes.

Example Bash:

```bash
start=$(date +%s)

flock 9

acquired=$(date +%s)
waited=$((acquired - start))

logger -t job \
    "lock_wait_seconds=$waited"
```

Increasing wait time is an early indicator of contention.

If a five-minute job waits four minutes for its lock, capacity is nearly exhausted.

---

## Monitoring lock hold time

The job duration is often the lock hold time if the whole job is protected.

Log:

```text
lock_acquired_at
lock_released_at
duration
```

For finer-grained locking, record critical-section duration separately.

This helps determine whether the expensive part actually needs serialization.

---

## Alert on skipped-run thresholds

A single skip may be normal.

Policy:

```text
alert if three consecutive runs skip
```

or:

```text
alert if last_success > 30 minutes
```

is more useful than alerting every lock conflict.

Monitor business freshness, not raw lock events alone.

---

## `last_success` is often the best simple signal

If a job should succeed at least once every 15 minutes, record:

```text
/var/lib/job/last-success
```

Then alert if:

```text
mtime older than 30 minutes
```

This naturally handles:

```text
one skipped run
small delays
temporary lock contention
```

without generating unnecessary alerts.

---

## Locks and daylight saving time

Cron can run a wall-clock time twice or skip it around DST transitions depending on implementation and timezone.

A lock may prevent concurrent duplicate runs if the first is still active.

But if the first finishes before the repeated wall-clock occurrence, the second can still run.

If "once per calendar day" matters, use durable idempotency keyed by date.

Locking protects concurrency.

It does not enforce calendar uniqueness.

---

## Locks and manual reruns

An administrator may run:

```bash
sudo /opt/jobs/backup
```

while cron is already running it.

If locking exists inside the script, manual execution follows the same policy.

If locking exists only in the crontab:

```cron
flock ... /opt/jobs/backup
```

manual execution bypasses it.

This is a strong argument for placing essential locking inside the job wrapper itself.

---

## Locks and deployment scripts

A deployment may invoke:

```bash
/opt/jobs/migrate
```

while cron also invokes the same migration.

Again, internal locking ensures every entry point coordinates.

Concurrency rules should live as close as possible to the shared operation.

---

## Locks should be documented

A script header can state:

```text
Lock: /run/company-report.lock
Policy: skip if another instance is running
Scope: one host
Expected max runtime: 20 minutes
Timeout: 30 minutes
```

This helps operators understand:

```text
why a run skipped
whether a lock is global
what runtime is abnormal
```

Concurrency behavior is part of the job's interface.

---

## A README for a scheduled job

Useful operational metadata:

```text
Schedule
Execution user
Lock path
Lock policy
Timeout
Expected runtime
Inputs
Outputs
Last-success indicator
Logs
Manual run command
Safe retry behavior
Idempotency model
Dependencies
```

This is especially valuable when cron tasks outlive their original authors.

---

## Avoid implicit lock names from script filenames

A wrapper may derive:

```bash
LOCK="/run/$(basename "$0").lock"
```

This is convenient but can fail if the same logical script is invoked through different symlink names.

Example:

```text
/usr/local/bin/report
/usr/local/bin/daily-report -> report
```

Now two invocations can derive different locks.

Explicit lock identity is safer for critical jobs.

---

## Locks around symlink-switched deployments

Suppose:

```text
/srv/app/current
```

points to a release directory.

A cron job resolves:

```text
/srv/app/current/bin/job
```

while a deployment changes the symlink.

Possible outcomes depend on when path resolution occurs.

Locking the job does not automatically coordinate deployment.

If deployment and scheduled work conflict, they need a shared deployment/maintenance lock.

Example:

```text
deploy lock
job reads same deploy lock
```

or stop/suspend scheduler during atomic release switching.

Concurrency exists between different kinds of operations, not only identical jobs.

---

## Deployment and cron race

Timeline:

```text
12:00 cron starts app job from release A
12:01 deploy switches current -> release B
12:02 deploy deletes release A
12:03 cron tries to load another file from release A
```

The running process may fail.

Solutions include:

```text
do not delete old release until jobs finish
versioned absolute paths
deployment lock
service manager lifecycle
self-contained process startup
```

Cron concurrency analysis should include deployment processes.

---

## Log rotation and cron race

A cron job may write:

```text
/var/log/job.log
```

while logrotate renames the file.

If the cron process opens the file only at invocation:

```cron
>> /var/log/job.log
```

each new run follows the current pathname.

A long-running process can keep writing to a renamed inode if it holds the descriptor open.

This is normal Unix behavior.

For short cron jobs, usually harmless.

For long-running scheduled processes, understand log rotation interaction.

---

## Backup versus cleanup race

Cron:

```cron
0 2 * * * backup
30 2 * * * delete-old-files
```

If backup takes more than 30 minutes, cleanup may delete files while backup scans them.

A shared lock can encode:

```text
backup and cleanup must not overlap
```

Use one resource lock:

```text
/run/archive-maintenance.lock
```

for both scripts.

Lock names should represent the shared resource, not necessarily the individual command.

This is an important design improvement.

---

## Resource-oriented locks

Instead of:

```text
backup.lock
cleanup.lock
```

use:

```text
archive-store.lock
```

if both operations mutate:

```text
archive store
```

Now all operations that require exclusive access to that resource coordinate through the same lock.

This is closer to correct concurrency modeling.

---

## Reader-writer locks

Some workloads allow:

```text
many readers
one writer
```

Traditional shell `flock` can request shared or exclusive locks.

Examples:

```bash
flock -s FILE COMMAND
```

for shared lock.

```bash
flock -x FILE COMMAND
```

for exclusive lock.

Multiple shared holders can coexist.

An exclusive holder conflicts with both shared and exclusive locks.

This can be useful when:

```text
reports read a stable dataset
maintenance modifies dataset
```

if all participants use the same lock protocol.

Check installed `flock` semantics through:

```bash
man flock
```

---

## Shared-lock example

Reader:

```bash
#!/bin/sh

exec 9>/run/dataset.lock
flock -s 9

read_dataset
```

Writer:

```bash
#!/bin/sh

exec 9>/run/dataset.lock
flock -x 9

update_dataset
```

Several readers may run simultaneously.

The writer waits until readers release the lock.

This is more sophisticated than most cron jobs need, but it shows advisory locking is not limited to simple mutexes.

---

## Shared locks can starve writers depending on behavior

If readers continuously arrive, assumptions about writer fairness should be verified.

If a writer must run within a strict deadline, a custom reader-writer lock policy or maintenance window may be more appropriate.

Do not build critical fairness assumptions on undocumented scheduler/lock ordering.

---

## File locking versus application locking

File lock strengths:

```text
simple
kernel-managed
excellent on one host
easy in shell
```

Limitations:

```text
local coordination domain
all participants must cooperate
does not encode business identity automatically
network filesystem semantics vary
```

Application/database lock strengths:

```text
can coordinate multiple hosts
can align with business resource
can participate in transactions
```

Limitations:

```text
more complexity
dependency on application infrastructure
lease/recovery semantics
```

Choose the lock nearest the resource and topology being protected.

---

## A decision guide

If one script on one Linux host must not overlap:

```text
use flock
```

If multiple scripts on one host share one filesystem resource:

```text
use a common flock lock
```

If multiple servers modify one database:

```text
use database constraints/transactions/advisory locks
```

If jobs need retries, history, concurrency limits, and durable backlog:

```text
use a queue/job system
```

If workload runs in Kubernetes:

```text
prefer Kubernetes CronJob/job controls
```

If one periodic reconciliation can safely skip intermediate runs:

```text
use non-blocking lock
```

If every scheduled event must be preserved:

```text
do not represent the event only as a cron tick
```

---

## A production readiness checklist

Before deploying a recurring job, answer:

```text
How long does it normally run?
What is the worst observed runtime?
Can two runs overlap safely?
What state do they share?
Is the operation idempotent?
What happens on retry?
What happens after partial success?
What happens after process crash?
Should a new run wait, skip, or replace?
Is the lock local or global?
Can another host run the same job?
Can manual execution bypass the lock?
Does deployment conflict with the job?
Is there a timeout?
How are skipped runs monitored?
How is last success monitored?
Can the job resume after interruption?
Does the database enforce uniqueness?
Are output files published atomically?
```

If those questions have clear answers, the scheduling design is usually robust.

---

## Final perspective

Cron does not serialize jobs.

It does not know which processes belong to the same logical task.

It does not know that two scripts share a backup directory.

It does not know that a second invoice email is a duplicate.

It does not know that a slow old report should not overwrite a newer one.

Cron only knows time.

Reliability comes from the layers around the schedule.

For single-host mutual exclusion, Linux `flock` is usually the best starting point:

```bash
flock -n /run/job.lock /opt/jobs/job
```

or descriptor-style locking inside the job.

Kernel-managed locking avoids stale lock-file state and naturally releases on process death.

But locks are not enough.

Reliable scheduled work also depends on:

```text
idempotency
atomic publication
durable state
database constraints
timeouts
retry policy
monitoring
correct lock scope
multi-host coordination
safe restart behavior
```

The strongest design does not assume:

```text
cron will run exactly once
```

It assumes:

```text
a run can be delayed
a run can be skipped
a run can happen twice
a process can crash
a server can reboot
an operator can rerun it
another host can start the same workflow
```

and still keeps the system correct.

That is the difference between a command that happens to be scheduled and a reliable scheduled job.
