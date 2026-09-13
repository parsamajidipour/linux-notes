# Cron Environment and Shell Behavior

A cron entry looks deceptively similar to a command typed in a terminal. That similarity is one of the main reasons cron jobs fail in production.

A command launched from an interactive shell inherits a large execution context: a login environment, a current directory chosen by the user, shell startup files, language and runtime managers, SSH agent variables, terminal settings, aliases, functions, locale configuration, and often an expanded `PATH`. A command launched by cron usually receives a much smaller and more controlled environment. It is also non-interactive, normally has no terminal attached, and is typically executed through a shell selected by the cron implementation or by variables in the crontab.

The important lesson is that a cron schedule answers only one question:

> When should this command be considered for execution?

It does not guarantee that the command runs in the same environment in which it was tested manually.

A reliable cron job must therefore be designed around an explicit execution context. Paths, shell semantics, working directory, credentials, locale, runtime configuration, output handling, and dependencies should be intentional rather than inherited accidentally.

This chapter focuses on that execution context.

---

## A useful mental model

Consider the following entry in a user crontab:

```cron
*/5 * * * * /opt/jobs/check-api.sh
```

It is tempting to imagine cron doing the equivalent of this:

```bash
/opt/jobs/check-api.sh
```

inside the user's normal terminal.

That is not the right model.

A more useful conceptual model is:

```text
cron daemon
    |
    +-- decides that the time expression matches
    |
    +-- establishes credentials and a limited environment
    |
    +-- invokes a shell
          |
          +-- shell parses the command text
                |
                +-- /opt/jobs/check-api.sh
```

Depending on the cron implementation and operating system, PAM, mail handling, SELinux context transitions, resource limits, or implementation-specific setup may also be involved. The important part is that cron is not reusing the user's terminal session.

That difference explains failures such as:

```text
command not found
module not found
permission denied
could not connect to agent
unable to open display
database configuration missing
wrong language or encoding
file not found
works manually but fails from cron
```

Those errors often have nothing to do with the schedule itself.

A practical debugging question is therefore not:

> Does this command work?

but:

> Does this command work under the same user, shell, environment, directory, and non-interactive conditions that cron will use?

That is a much more precise question.

---

## Inspecting the environment before changing anything

Before trying to fix a failing cron job, capture what cron actually sees.

Create a temporary diagnostic job:

```cron
* * * * * /usr/bin/env > /tmp/cron-env.txt 2>&1
```

After one minute:

```bash
cat /tmp/cron-env.txt
```

Now compare it with an interactive shell:

```bash
env | sort > /tmp/interactive-env.txt
sort /tmp/cron-env.txt > /tmp/cron-env.sorted.txt
diff -u /tmp/cron-env.sorted.txt /tmp/interactive-env.txt
```

A real system may show differences similar to these:

```diff
-SHELL=/bin/sh
-PATH=/usr/bin:/bin
+SHELL=/bin/bash
+PATH=/home/alice/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
+SSH_AUTH_SOCK=/run/user/1000/keyring/ssh
+NVM_DIR=/home/alice/.nvm
+VIRTUAL_ENV=/home/alice/projects/api/.venv
+TERM=xterm-256color
```

The exact values are implementation- and distribution-dependent. The comparison is what matters.

A second useful diagnostic job records identity, working directory, shell-visible limits, and file descriptors:

```cron
* * * * * /bin/sh -c '{
    echo "DATE=$(date -Is)";
    echo "UID=$(id -u)";
    echo "USER=$(id -un)";
    echo "GROUPS=$(id -Gn)";
    echo "PWD=$PWD";
    echo "SHELL=$SHELL";
    echo "PATH=$PATH";
    echo "--- limits ---";
    ulimit -a;
    echo "--- fds ---";
    ls -l /proc/$$/fd;
} > /tmp/cron-context.txt 2>&1'
```

This kind of evidence is much more useful than repeatedly editing the crontab and waiting to see whether the problem disappears.

---

## Cron is not a login shell

When a user logs in through a terminal, SSH, a display manager, or another login mechanism, shell startup behavior may load files such as:

```text
/etc/profile
~/.profile
~/.bash_profile
~/.bash_login
~/.bashrc
```

Which files are read depends on the shell and whether it is a login shell or an interactive shell.

Cron does not normally launch a login shell for each job. Therefore, assumptions such as these are unsafe:

```bash
export PATH="$HOME/.local/bin:$PATH"
source "$HOME/.nvm/nvm.sh"
alias python=python3
export APP_ENV=production
```

if they exist only in `.bashrc` or `.profile`.

For example, suppose a developer has this in `~/.bashrc`:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

and installs a tool here:

```text
/home/alice/.local/bin/report-generator
```

In a terminal:

```bash
report-generator --daily
```

works.

The crontab entry:

```cron
0 2 * * * report-generator --daily
```

may fail with:

```text
/bin/sh: 1: report-generator: not found
```

The robust form is:

```cron
0 2 * * * /home/alice/.local/bin/report-generator --daily
```

or, if the application legitimately depends on a custom path:

```cron
PATH=/home/alice/.local/bin:/usr/local/bin:/usr/bin:/bin

0 2 * * * report-generator --daily
```

The first form is generally easier to audit because the executable being invoked is explicit.

This distinction becomes even more important when commands are executed with elevated privileges. A root cron job that depends on a user-controlled `PATH` can turn a configuration mistake into a privilege-escalation vulnerability.

---

## The shell used by cron

Traditional cron implementations execute the command part of a crontab entry using a shell. On many Unix-like systems the default is `/bin/sh`, although details vary by implementation.

A typical conceptual invocation is close to:

```bash
/bin/sh -c 'command from crontab'
```

This matters because `/bin/sh` is not necessarily Bash.

On Debian and Ubuntu systems, for example:

```bash
ls -l /bin/sh
```

often shows:

```text
/bin/sh -> dash
```

A command tested in Bash may therefore fail when cron evaluates it under a POSIX shell.

Consider:

```cron
* * * * * [[ -f /tmp/ready ]] && echo ready >> /tmp/state.log
```

`[[ ... ]]` is a Bash construct. If cron uses `/bin/sh` and `/bin/sh` is `dash`, the job can fail.

A portable version is:

```cron
* * * * * [ -f /tmp/ready ] && echo ready >> /tmp/state.log
```

Another example:

```cron
* * * * * echo {1..5} >> /tmp/numbers.log
```

Bash performs brace expansion and produces:

```text
1 2 3 4 5
```

A POSIX shell that does not implement Bash brace expansion may write:

```text
{1..5}
```

Similarly, Bash arrays are not portable:

```bash
servers=(api-1 api-2 api-3)
```

Neither is process substitution:

```bash
diff <(command-a) <(command-b)
```

If a cron job needs Bash, state that requirement.

A crontab can often define:

```cron
SHELL=/bin/bash
```

and then use Bash semantics:

```cron
SHELL=/bin/bash

*/10 * * * * [[ -f /run/app.ready ]] && /opt/app/check.sh
```

A more explicit design is to keep shell logic in a script:

```bash
#!/usr/bin/env bash
set -euo pipefail

if [[ -f /run/app.ready ]]; then
    /opt/app/check.sh
fi
```

with a simple cron entry:

```cron
*/10 * * * * /opt/jobs/check-ready.sh
```

This has several advantages. The script can be syntax-checked, tested manually, reviewed in Git, instrumented, and reused outside cron.

---

## The `SHELL` variable changes parsing, not every program's interpreter

A subtle point is that the shell used to interpret the crontab command and the interpreter used by a script are separate concerns.

Suppose the crontab contains:

```cron
SHELL=/bin/bash

* * * * * /opt/jobs/collect.py
```

and `collect.py` begins with:

```python
#!/usr/bin/python3
```

The shell parses the command line, finds `/opt/jobs/collect.py`, and executes it. The kernel then reads the shebang and starts `/usr/bin/python3`.

Changing `SHELL` to Bash does not make Python scripts run "inside Bash".

Likewise:

```bash
#!/usr/bin/env python3
```

asks `/usr/bin/env` to locate `python3` through `PATH`. That makes the script more portable between environments, but it also makes the interpreter selection dependent on the runtime `PATH`.

For scheduled production jobs, an explicit interpreter can sometimes be safer:

```bash
#!/opt/reporting/.venv/bin/python
```

or the cron entry can invoke it directly:

```cron
0 3 * * * /opt/reporting/.venv/bin/python /opt/reporting/report.py
```

The correct choice depends on deployment strategy. The key is to understand which component performs interpreter selection.

---

## `PATH` is one of the most common cron failures

`PATH` is a colon-separated list of directories searched when a command name does not contain `/`.

For example:

```bash
curl
```

requires a path search.

This does not:

```bash
/usr/bin/curl
```

Suppose a backup script contains:

```bash
#!/bin/sh

tar -czf backup.tar.gz /srv/app
aws s3 cp backup.tar.gz s3://company-backups/
```

It works in the administrator's shell because:

```bash
command -v tar
command -v aws
```

returns:

```text
/usr/bin/tar
/usr/local/bin/aws
```

A restricted cron path may include `/usr/bin` but not `/usr/local/bin`. The first command succeeds and the second fails.

There are several ways to solve the problem.

Use absolute paths:

```bash
#!/bin/sh

/usr/bin/tar -czf /var/backups/app.tar.gz /srv/app
/usr/local/bin/aws s3 cp /var/backups/app.tar.gz s3://company-backups/
```

Define a controlled path in the script:

```bash
#!/bin/sh

PATH=/usr/local/bin:/usr/bin:/bin
export PATH

tar -czf /var/backups/app.tar.gz /srv/app
aws s3 cp /var/backups/app.tar.gz s3://company-backups/
```

Or define it in the crontab:

```cron
PATH=/usr/local/bin:/usr/bin:/bin

0 1 * * * /opt/jobs/backup.sh
```

For security-sensitive jobs, a short controlled `PATH` is preferable to copying a long interactive path containing user-writable directories.

This is risky:

```text
/home/alice/bin:/home/alice/.local/bin:/usr/local/bin:/usr/bin:/bin
```

for a privileged system job.

A root job should not casually search user-controlled directories for commands.

To inspect every command dependency of a shell script, one useful technique is:

```bash
grep -Eo '(^|[;&|[:space:]])[a-zA-Z0-9_./-]+' /opt/jobs/backup.sh
```

but static grep is only a rough aid. A better practical approach is to run the script in a reduced environment and trace failures:

```bash
env -i \
    HOME=/root \
    PATH=/usr/bin:/bin \
    SHELL=/bin/sh \
    /opt/jobs/backup.sh
```

If the script depends on hidden shell state, this test often reveals it immediately.

---

## The current working directory is not your project directory

Relative paths are another major source of cron bugs.

Imagine a project:

```text
/opt/reporting/
├── config.yaml
├── reports/
└── scripts/
    └── daily.sh
```

with:

```bash
#!/bin/sh

python3 generate.py --config ../config.yaml > ../reports/today.txt
```

A developer tests it like this:

```bash
cd /opt/reporting/scripts
./daily.sh
```

and it works.

The cron entry is:

```cron
0 6 * * * /opt/reporting/scripts/daily.sh
```

Now the script may fail because its relative paths are evaluated from cron's current working directory, not necessarily from `/opt/reporting/scripts`.

Never use the successful manual test as proof that relative paths are safe.

A robust script can establish its own location:

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_DIR="$(dirname "$SCRIPT_DIR")"

cd "$PROJECT_DIR"

/usr/bin/python3 scripts/generate.py \
    --config config.yaml \
    > reports/today.txt
```

For POSIX `sh`, a simpler pattern may be:

```bash
#!/bin/sh
set -eu

cd /opt/reporting
/usr/bin/python3 scripts/generate.py \
    --config config.yaml \
    > reports/today.txt
```

For production automation, an explicit fixed directory is often easier to reason about than clever path discovery.

The crontab itself can also change directory:

```cron
0 6 * * * cd /opt/reporting && /usr/bin/python3 scripts/generate.py --config config.yaml
```

This works, but once a command becomes complex enough to need multiple operators, redirections, or error handling, moving the logic into a script usually improves maintainability.

You can prove the current directory cron sees with:

```cron
* * * * * /bin/pwd > /tmp/cron-pwd.txt
```

Do not assume. Measure.

---

## `HOME`, `LOGNAME`, and user identity

Cron jobs are associated with a user context.

A user crontab normally runs entries as that user. System crontabs and files in locations such as `/etc/cron.d/` may include an explicit username field, depending on the implementation.

The execution environment commonly contains variables related to that account, such as:

```text
HOME=/home/alice
LOGNAME=alice
USER=alice
```

but exact variable names and initialization behavior can vary.

Programs frequently use `HOME` to locate configuration:

```text
~/.aws/credentials
~/.config/
~/.ssh/
~/.docker/
~/.local/
```

That creates an important difference between these two commands:

```bash
sudo /opt/jobs/publish.sh
```

and:

```cron
0 * * * * /opt/jobs/publish.sh
```

in root's crontab.

The first command may preserve or reset environment variables according to `sudo` policy. The second is launched by cron under root's cron context. They are not equivalent tests.

To test the actual account environment more closely, use:

```bash
sudo -u alice env -i \
    HOME=/home/alice \
    USER=alice \
    LOGNAME=alice \
    PATH=/usr/bin:/bin \
    SHELL=/bin/sh \
    /opt/jobs/publish.sh
```

If a job depends on a file such as:

```text
/home/alice/.config/myapp/config.toml
```

it is better for the application or script to reference the intended configuration explicitly when possible:

```bash
/usr/local/bin/myapp \
    --config /etc/myapp/reporting.toml
```

This avoids coupling a production automation task to a user's interactive home-directory state.

---

## Variables can be defined directly in a crontab

Crontabs support environment assignments in addition to schedule lines.

For example:

```cron
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin
APP_ENV=production
REPORT_DIR=/var/lib/reporting

15 2 * * * /opt/reporting/daily.sh
```

The assignment lines are not scheduled jobs. They define environment values used by following entries according to the cron implementation's parsing rules.

This is useful for small, stable values.

It is not a good place for large application configurations or secrets.

For example:

```cron
DB_PASSWORD=SuperSecretPassword
```

may expose sensitive data to anyone who can read the crontab, to backups, configuration snapshots, administrative tooling, or command output used during troubleshooting.

A better design is usually to place secrets in a file with controlled ownership and permissions:

```bash
sudo install -o root -g reporting -m 0640 \
    /dev/null /etc/reporting/reporting.env
```

Then store:

```text
DB_HOST=db.internal
DB_USER=reporter
DB_PASSWORD=...
```

and have the application load that file through a secure configuration mechanism.

Be careful with shell sourcing:

```bash
. /etc/reporting/reporting.env
```

is not a generic environment-file parser. It executes the file as shell code. If the file is writable by an untrusted user, sourcing it is equivalent to executing attacker-controlled commands.

The security model of configuration loading matters.

---

## Inline variable assignments have shell semantics

This cron entry:

```cron
0 * * * * APP_ENV=production /opt/app/run-report
```

uses a shell-style environment assignment for that command.

It is conceptually similar to:

```bash
APP_ENV=production /opt/app/run-report
```

The variable exists in the environment of the launched command but does not permanently modify the daemon or the user's shell.

You can observe this:

```bash
APP_ENV=production /usr/bin/env | grep '^APP_ENV='
```

returns:

```text
APP_ENV=production
```

while:

```bash
echo "$APP_ENV"
```

in the parent shell may remain empty.

This is useful when one job needs a value that should not apply to other entries:

```cron
0 2 * * * APP_MODE=daily /opt/app/report.sh
0 3 * * 1 APP_MODE=weekly /opt/app/report.sh
```

If the values become numerous, move them into a configuration file or wrapper script. A crontab should remain auditable at a glance.

---

## Quoting rules are shell rules

Once cron hands the command string to a shell, normal shell parsing becomes relevant.

Compare:

```cron
* * * * * echo $HOME >> /tmp/home.log
```

with:

```cron
* * * * * echo '$HOME' >> /tmp/home.log
```

The first expands the variable. The second writes the literal text:

```text
$HOME
```

Double quotes allow parameter expansion:

```cron
* * * * * echo "$HOME" >> /tmp/home.log
```

This matters when values contain spaces.

Suppose:

```cron
REPORT_DIR=/var/lib/company reports
```

This is already problematic because assignment parsing and whitespace rules can become confusing. A safer approach is to keep configuration values out of crontab when quoting becomes complex.

Similarly:

```cron
0 2 * * * /opt/bin/report --title Daily Security Report
```

passes several arguments:

```text
--title
Daily
Security
Report
```

If the program expects a single title:

```cron
0 2 * * * /opt/bin/report --title "Daily Security Report"
```

is correct.

When cron lines become dense with nested quotes:

```cron
* * * * * /bin/sh -c 'something "$(another-command | awk '\''{print $2}'\'')"'
```

the design has already become harder to review than necessary.

Move it into a script.

Cron is a scheduler, not a programming language.

---

## Percent signs deserve special attention

Traditional crontab syntax gives `%` a special meaning in the command portion. An unescaped percent sign may be transformed into a newline, with the remainder delivered as standard input to the command. Exact behavior should be checked against the cron implementation in use.

This can surprise users writing date commands such as:

```cron
* * * * * date +%F >> /tmp/date.log
```

On implementations with traditional `%` handling, this can break unexpectedly.

Escaping the percent signs is the usual solution:

```cron
* * * * * date +\%F >> /tmp/date.log
```

The safest design for nontrivial formatting is again to place the command in a script:

```bash
#!/bin/sh
/usr/bin/date +%F >> /tmp/date.log
```

with:

```cron
* * * * * /opt/jobs/write-date.sh
```

This reduces the number of syntax layers that must be mentally evaluated.

A cron command may be parsed by:

```text
crontab parser
    -> shell parser
        -> utility-specific parser
```

Every additional layer of quoting or escaping increases the chance of an operational mistake.

---

## Interactive shell features are absent

Cron jobs are non-interactive.

They do not normally have an attached terminal. Commands that prompt for input can block or fail.

This script is unsuitable:

```bash
#!/bin/bash

read -p "Continue? [y/N] " answer
if [[ "$answer" == "y" ]]; then
    /opt/app/deploy
fi
```

When started from a terminal, a user can answer.

When started from cron, there is no human waiting at a terminal.

Similarly:

```bash
sudo some-command
```

may fail with an error like:

```text
sudo: a terminal is required to read the password
```

or:

```text
sudo: no tty present and no askpass program specified
```

depending on system configuration and sudo version.

A scheduled task should not depend on an interactive password prompt.

If a command needs elevated privileges, choose the execution account deliberately:

```text
root crontab
system crontab with explicit user
dedicated privileged helper
systemd service/timer with controlled permissions
```

rather than embedding an interactive `sudo`.

The same problem occurs with:

```text
ssh host
gpg
mysql
psql
docker login
cloud CLIs
secret stores
```

when those programs expect a password, PIN, confirmation prompt, or interactive authentication flow.

Automation requires non-interactive credentials and explicit failure behavior.

---

## There is normally no terminal

A terminal is more than a place where text appears. It provides a TTY device and terminal-specific behavior.

In an interactive shell:

```bash
tty
```

might print:

```text
/dev/pts/2
```

Under cron:

```cron
* * * * * /usr/bin/tty > /tmp/cron-tty.txt 2>&1
```

often results in:

```text
not a tty
```

Programs sometimes change behavior when stdout is not connected to a terminal. They may disable colors, buffering may change, progress bars may disappear, prompts may fail, and line formatting can differ.

A Python example makes buffering visible.

Consider:

```python
import time

for i in range(10):
    print(i)
    time.sleep(5)
```

When output is redirected to a file, buffering can make the log appear delayed.

A scheduled command can use unbuffered Python mode:

```cron
* * * * * /usr/bin/python3 -u /opt/jobs/progress.py >> /var/log/progress.log 2>&1
```

or the application can explicitly flush output.

Do not diagnose a job as "not running" solely because its output file is not updating immediately.

Observe the process:

```bash
pgrep -af progress.py
```

and inspect open files:

```bash
sudo lsof -p PID
```

---

## Shell startup files should not be forced without understanding them

A common workaround for cron failures is:

```cron
* * * * * bash -l -c '/opt/jobs/task.sh'
```

or:

```cron
* * * * * source ~/.bashrc && command
```

This can make a job appear to work, but it also imports a large amount of interactive user configuration into an automation context.

A `.bashrc` may contain:

```bash
alias rm='rm -i'
stty ...
bind ...
echo ...
nvm initialization
prompt setup
terminal detection
interactive-only functions
network calls
```

Some files guard interactive configuration:

```bash
case $- in
    *i*) ;;
      *) return;;
esac
```

which means sourcing them from a non-interactive shell may not even load the expected variables.

Instead of making cron imitate a full login session, identify the exact dependency.

If the only missing value is:

```bash
PATH="$HOME/.local/bin:$PATH"
```

set the required path explicitly.

If the application needs:

```bash
APP_ENV=production
```

define that configuration intentionally.

If a runtime manager is required, consider whether a fixed runtime path would be more deterministic.

Automation becomes more reliable as hidden dependencies are removed.

---

## Runtime managers are frequent sources of hidden state

Tools such as:

```text
nvm
pyenv
rbenv
asdf
SDKMAN
virtualenv activation scripts
conda
```

are often initialized from shell startup files.

A Node.js developer may have:

```bash
node --version
```

return:

```text
v24.x
```

in the terminal because `.bashrc` loads `nvm`.

Cron may see:

```text
node: not found
```

because `nvm` is a shell function and its installation path is not in cron's `PATH`.

One fragile workaround is:

```cron
0 * * * * . /home/alice/.nvm/nvm.sh && node /opt/app/job.js
```

A more deterministic deployment can reference the selected runtime directly, for example:

```cron
0 * * * * /opt/node/bin/node /opt/app/job.js
```

or package the application so its runtime path is stable.

Python virtual environments have the same issue.

Interactive workflow:

```bash
cd /opt/reporting
source .venv/bin/activate
python report.py
```

Scheduled workflow:

```cron
0 5 * * * * /opt/reporting/.venv/bin/python /opt/reporting/report.py
```

Activation is often unnecessary when the interpreter inside the virtual environment is invoked directly.

Verify:

```bash
/opt/reporting/.venv/bin/python -c 'import sys; print(sys.executable)'
```

Expected:

```text
/opt/reporting/.venv/bin/python
```

This is easier to reproduce and audit than relying on shell activation.

---

## `#!/usr/bin/env` is convenient but path-dependent

A shebang such as:

```bash
#!/usr/bin/env python3
```

is common because the interpreter may live in different directories on different systems.

The execution chain is approximately:

```text
kernel
  -> /usr/bin/env
      -> search PATH for python3
          -> run selected interpreter
```

This means interpreter selection depends on `PATH`.

Under an interactive shell:

```bash
command -v python3
```

could return:

```text
/home/alice/.pyenv/shims/python3
```

while cron might resolve:

```text
/usr/bin/python3
```

The script can therefore run successfully in both contexts while using different Python versions and different installed modules.

That is more dangerous than a simple "command not found" because the job may execute with subtly different behavior.

Inspect the runtime explicitly:

```python
import os
import sys

print("python:", sys.executable)
print("version:", sys.version)
print("PATH:", os.environ.get("PATH"))
```

For production jobs, deterministic interpreter selection is usually worth the small loss of portability.

---

## Locale differences can change program behavior

Locale variables influence character encoding, sorting, date formatting, numeric formatting, regular-expression behavior, and application translations.

Relevant variables include:

```text
LANG
LC_ALL
LC_CTYPE
LC_COLLATE
LC_TIME
LC_NUMERIC
```

An interactive shell might contain:

```bash
locale
```

with output such as:

```text
LANG=en_US.UTF-8
LC_CTYPE="en_US.UTF-8"
LC_TIME="en_US.UTF-8"
...
```

A scheduled process may see a smaller environment.

This can produce surprising failures.

Consider a script processing UTF-8 filenames:

```bash
find /srv/uploads -type f | sort
```

Sorting behavior can change between locales.

A Python application may also behave differently if encoding assumptions are implicit.

For deterministic text-processing jobs, it can be useful to define locale explicitly:

```cron
LANG=C.UTF-8
LC_ALL=C.UTF-8

0 4 * * * /opt/jobs/build-index.sh
```

or, for byte-oriented predictable sorting:

```cron
LC_ALL=C

0 4 * * * /opt/jobs/build-index.sh
```

The correct locale depends on the application.

Do not blindly set `LC_ALL=C` for software that needs Unicode-aware behavior. The point is to make the expected locale explicit.

A useful test is:

```cron
* * * * * /usr/bin/locale > /tmp/cron-locale.txt 2>&1
```

Compare it with the interactive session.

---

## Timezone is part of the execution environment too

Cron scheduling and application timezone are related but separate ideas.

The scheduler determines when a command starts according to the cron implementation and configured scheduling timezone behavior.

Once the program starts, it may independently use timezone information from:

```text
system timezone
TZ environment variable
application configuration
database session timezone
container configuration
```

Imagine a reporting job scheduled for midnight:

```cron
0 0 * * * /opt/reports/daily.sh
```

Inside it:

```bash
date +%F
```

The program may format a date using a timezone that differs from the scheduler's intended business timezone.

For applications spanning multiple regions, record timezone explicitly in logs:

```bash
date -Is
date '+%Y-%m-%d %H:%M:%S %Z %z'
```

A robust reporting system should decide whether "daily" means:

```text
midnight UTC
midnight server-local time
midnight Muscat time
midnight customer-local time
```

and encode that decision explicitly.

Cron can start a process at the intended time while the application still calculates the wrong reporting period.

---

## `MAILTO` and job output

Traditional cron implementations commonly capture job output and may mail it to a configured recipient if local mail delivery is available.

A crontab can use:

```cron
MAILTO=ops@example.com
```

or, depending on the desired behavior:

```cron
MAILTO=""
```

to suppress mail in implementations that support it.

Many modern servers do not have a functional local mail transfer agent, so silently assuming cron mail provides monitoring is dangerous.

A better operational model is to route output intentionally.

For example:

```cron
0 2 * * * /opt/jobs/backup.sh >> /var/log/backup.log 2>&1
```

This combines stdout and stderr.

Separate files are sometimes better:

```cron
0 2 * * * /opt/jobs/backup.sh \
    >> /var/log/backup.out.log \
    2>> /var/log/backup.err.log
```

However, long-running applications should generally integrate with system logging rather than grow unmanaged files forever.

A wrapper can use `logger`:

```bash
#!/bin/sh

if /opt/app/backup; then
    /usr/bin/logger -t app-backup "backup completed"
else
    rc=$?
    /usr/bin/logger -t app-backup "backup failed rc=$rc"
    exit "$rc"
fi
```

Then inspect:

```bash
journalctl -t app-backup
```

on systems where the journal receives those messages.

Logging is part of execution design, not an afterthought.

---

## Redirection order changes meaning

Shell redirection syntax is processed from left to right.

These two commands are not always equivalent:

```bash
command > file 2>&1
```

and:

```bash
command 2>&1 > file
```

The first means:

```text
stdout -> file
stderr -> current stdout -> file
```

so both streams go to the file.

The second means:

```text
stderr -> current stdout
stdout -> file
```

so stderr may still go wherever stdout was pointing before the redirection.

This matters in cron because output may otherwise be mailed, discarded, or captured by another mechanism.

For most simple cron logging:

```cron
0 * * * * /opt/jobs/task.sh >> /var/log/task.log 2>&1
```

is the intended form.

If the job produces sensitive output, permissions on the log matter:

```bash
sudo install -o root -g adm -m 0640 /dev/null /var/log/task.log
```

Do not redirect secrets into world-readable files.

---

## Pipelines can hide failures

This cron entry looks reasonable:

```cron
0 * * * * /opt/app/export | gzip > /var/backups/export.gz
```

But pipeline exit semantics matter.

Under traditional POSIX shell behavior, the status of a pipeline is often the status of the last command. If `/opt/app/export` fails but `gzip` exits successfully after receiving incomplete input, the overall command can appear successful.

Bash provides `pipefail`:

```bash
set -o pipefail
```

A robust Bash wrapper could be:

```bash
#!/usr/bin/env bash
set -euo pipefail

/opt/app/export | /usr/bin/gzip > /var/backups/export.gz
```

and cron becomes:

```cron
0 * * * * /opt/jobs/export-backup.sh
```

This is another example of why shell behavior belongs in a script when correctness matters.

Be cautious with `set -e` as well. Its semantics have exceptions and can surprise users inside conditionals, pipelines, and compound commands. It is useful when understood, not a substitute for explicit error handling.

---

## Exit status is the machine-readable result

Unix processes return an exit status.

Conventionally:

```text
0   success
non-zero   failure or another condition
```

Cron itself does not magically understand whether an application achieved a business goal. It can launch the command; operational tooling must decide how to observe the result.

Test a command:

```bash
/opt/jobs/task.sh
echo $?
```

A wrapper can preserve the result:

```bash
#!/bin/sh

/opt/app/reconcile
rc=$?

if [ "$rc" -ne 0 ]; then
    /usr/bin/logger -p user.err -t reconcile "failed rc=$rc"
fi

exit "$rc"
```

A common mistake is:

```bash
/opt/app/reconcile
echo "done"
```

If `reconcile` exits with status `1` but `echo` succeeds, the script's final status may be `0`.

A corrected version:

```bash
#!/bin/sh
set -e

/opt/app/reconcile
echo "done"
```

or explicit handling:

```bash
#!/bin/sh

if /opt/app/reconcile; then
    echo "done"
else
    rc=$?
    echo "failed rc=$rc" >&2
    exit "$rc"
fi
```

Scheduled automation should make failure observable.

---

## Shell builtins versus external commands

Not every command name corresponds to a file on disk.

Examples of shell builtins include, depending on shell:

```text
cd
export
read
umask
ulimit
```

Running:

```bash
command -v cd
```

may report:

```text
cd
```

because `cd` changes the current directory of the shell process itself. An external program cannot change its parent's current directory.

That explains why a cron entry like this uses shell semantics:

```cron
0 * * * * cd /opt/app && ./job.sh
```

The shell performs `cd`, then conditionally launches `./job.sh`.

Likewise:

```cron
0 * * * * export APP_ENV=production; /opt/app/job
```

works because `export` is interpreted by the shell.

However, this style quickly becomes hard to maintain.

Prefer:

```bash
#!/bin/sh
set -eu

export APP_ENV=production
cd /opt/app
exec ./job
```

and schedule:

```cron
0 * * * * /opt/jobs/run-app-job.sh
```

The use of `exec` replaces the wrapper shell with the target process. That can simplify process trees and signal handling, although it is not mandatory for every wrapper.

---

## `umask` affects files created by scheduled jobs

File permissions are influenced by both application-requested modes and the process umask.

A process creating a file with mode `0666` under umask `0022` typically ends up with:

```text
0644
```

because masked permission bits are removed.

Check the interactive shell:

```bash
umask
```

Possible output:

```text
0022
```

A cron job may inherit or receive a different effective umask depending on the implementation and execution setup.

Security-sensitive jobs should define expectations explicitly.

For example:

```bash
#!/bin/sh
set -eu

umask 0077
/usr/bin/openssl rand -hex 32 > /var/lib/app/private-token
```

This helps ensure newly created files are not group- or world-readable.

For shared operational artifacts:

```bash
umask 0027
```

may be more appropriate.

Do not solve permission problems by using:

```bash
chmod -R 777 ...
```

Cron exposes permission design problems quickly because automation lacks the informal assumptions of an administrator's shell.

---

## PAM and system-specific environment setup

On Linux distributions using PAM-aware cron implementations, the cron session may pass through PAM modules. That can influence:

```text
resource limits
environment values
access policy
session setup
security context
```

The exact behavior depends on distribution packaging and files such as:

```text
/etc/pam.d/cron
/etc/security/limits.conf
/etc/security/limits.d/
```

For this reason, statements like "cron always has exactly these variables" are too strong.

Inspect the system.

Useful commands include:

```bash
ps -ef | grep '[c]ron'
cron --version 2>/dev/null || true
crond --version 2>/dev/null || true
cat /etc/pam.d/cron 2>/dev/null
```

On RPM-based systems using Cronie, service names and configuration paths may differ from Debian-family systems.

The engineering principle is portable even when implementation details differ:

> Observe the actual execution environment of the target machine instead of assuming the behavior of a different distribution.

---

## Resource limits can differ from an interactive session

A scheduled job may run under limits affecting:

```text
open files
process count
locked memory
core file size
CPU time
address space
stack size
```

Inspect with:

```bash
ulimit -a
```

A diagnostic cron entry:

```cron
* * * * * /bin/sh -c 'ulimit -a' > /tmp/cron-ulimit.txt 2>&1
```

Compare with:

```bash
ulimit -a > /tmp/interactive-ulimit.txt
diff -u /tmp/cron-ulimit.txt /tmp/interactive-ulimit.txt
```

A database export tool opening many files may work interactively but fail under a lower `nofile` limit.

Do not immediately increase limits globally. First confirm:

```text
which process hits the limit
what limit applies
where it is configured
whether the job design is correct
```

When a scheduled service needs complex resource controls, a systemd service triggered by a systemd timer may provide a clearer place to configure limits than cron.

---

## SSH authentication usually depends on missing session state

A classic example is:

```cron
0 1 * * * git -C /srv/repo pull
```

The administrator says:

```bash
git -C /srv/repo pull
```

works manually.

Cron fails:

```text
Permission denied (publickey).
fatal: Could not read from remote repository.
```

The interactive terminal may contain:

```bash
echo "$SSH_AUTH_SOCK"
```

with a value like:

```text
/run/user/1000/keyring/ssh
```

This points to an SSH agent belonging to the login session.

Cron may have no `SSH_AUTH_SOCK`.

A scheduled job should not casually depend on an ephemeral desktop or SSH-login agent.

Possible designs include:

```text
a dedicated deploy key with tightly controlled permissions
a machine identity
a CI/CD system
a service credential
a dedicated agent with an intentionally managed lifecycle
```

If a private key is used, permissions matter:

```bash
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/id_ed25519
chmod 600 /home/deploy/.ssh/config
```

Host verification also matters. Disabling it with:

```text
StrictHostKeyChecking=no
```

may remove an important security property and should not be used as a reflexive cron fix.

The real problem is not "cron cannot use Git". The problem is that interactive authentication state was silently treated as part of the application.

---

## Graphical applications usually lack a desktop session

A user may try:

```cron
* * * * * notify-send "Backup complete"
```

and see nothing.

Desktop applications may depend on session-specific variables such as:

```text
DISPLAY
WAYLAND_DISPLAY
DBUS_SESSION_BUS_ADDRESS
XDG_RUNTIME_DIR
```

These belong to a graphical login session, not to cron's basic execution environment.

Hardcoding:

```cron
DISPLAY=:0
```

may work in a narrow X11 setup and still be architecturally fragile.

On modern Wayland desktops, session isolation is even more explicit.

Cron is best suited to background system or user tasks whose operation does not depend on a graphical session.

If a task should run as part of a user's graphical session, a desktop autostart mechanism or a per-user systemd unit may be a better fit.

---

## Containers and cron introduce another environment boundary

Running cron inside a container creates two layers of environment reasoning:

```text
container runtime environment
    -> cron daemon environment
        -> scheduled job environment
```

Suppose Docker Compose defines:

```yaml
environment:
  APP_ENV: production
  DB_HOST: db
```

A cron daemon started as the container's main process may inherit those variables, but whether jobs receive all of them can depend on the cron implementation and how it constructs job environments.

Do not assume container environment variables automatically propagate exactly as expected.

A diagnostic entry remains useful:

```cron
* * * * * /usr/bin/env | /usr/bin/sort > /tmp/cron-env.txt
```

For containerized workloads, it is often cleaner to schedule from outside the application container:

```text
host systemd timer
Kubernetes CronJob
orchestrator-native scheduled task
CI scheduler
```

rather than embed a traditional cron daemon inside every container.

The correct choice depends on the deployment model, but the environment boundary should be explicit.

---

## Docker commands from cron often fail for permissions, not Docker itself

A common entry:

```cron
0 4 * * * docker compose -f /srv/app/compose.yml exec -T app php artisan schedule:run
```

may fail because:

```text
docker: command not found
```

or:

```text
permission denied while trying to connect to the Docker daemon socket
```

Check the executable:

```bash
command -v docker
```

Then inspect the socket:

```bash
ls -l /var/run/docker.sock
```

and identity:

```bash
id
```

A user added to the `docker` group often has effectively root-equivalent control over the host through the Docker daemon. Treat that as a security decision, not a harmless permission fix.

A safer scheduled design may be:

```cron
0 4 * * * /usr/bin/docker compose -f /srv/app/compose.yml exec -T app php artisan schedule:run
```

under an account that is intentionally authorized to control Docker.

Also note the explicit Compose file path. Relying on:

```bash
docker compose ...
```

from an arbitrary working directory can cause Compose to discover the wrong project or no project at all.

---

## Database clients often depend on hidden credential files

An interactive backup might work:

```bash
pg_dump appdb > backup.sql
```

because the user's environment or home directory provides credentials.

Cron may fail with:

```text
password authentication failed
```

or may wait for input.

For PostgreSQL, one non-interactive mechanism can be a properly protected `.pgpass` file:

```text
hostname:port:database:username:password
```

with permissions such as:

```bash
chmod 600 ~/.pgpass
```

For MySQL/MariaDB, client option files can serve a similar purpose.

Do not put database passwords directly on the command line:

```bash
mysql -u app -pSecretPassword
```

because command-line arguments may be visible through process inspection and audit systems.

A backup job should define:

```text
which account runs it
where credentials come from
what permissions protect them
where backups are written
what happens on partial failure
how output is monitored
```

The cron line is only a small part of the system.

---

## A realistic example: a backup that works manually and fails from cron

Suppose this script exists:

```bash
#!/bin/bash

cd ~/projects/store
mysqldump store > backups/store.sql
tar -czf backups/store.tar.gz backups/store.sql
aws s3 cp backups/store.tar.gz s3://store-backups/
```

The user runs:

```bash
./backup.sh
```

successfully.

Cron:

```cron
0 2 * * * /home/alice/backup.sh
```

fails.

There are several hidden assumptions.

`~` depends on `HOME`.

`mysqldump` depends on `PATH`.

Database authentication may depend on home-directory configuration.

`tar` depends on `PATH`.

`aws` may live in `/usr/local/bin` or `~/.local/bin`.

AWS credentials may depend on:

```text
~/.aws/credentials
AWS_PROFILE
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

The script's error handling is weak. If `mysqldump` fails, a stale or empty file may still be compressed and uploaded.

A stronger version might be:

```bash
#!/usr/bin/env bash
set -euo pipefail

PATH=/usr/local/bin:/usr/bin:/bin
export PATH

umask 0077

PROJECT_DIR=/srv/store
BACKUP_DIR=/var/backups/store
STAMP=$(/usr/bin/date -u +%Y%m%dT%H%M%SZ)

SQL_FILE="$BACKUP_DIR/store-$STAMP.sql"
ARCHIVE="$BACKUP_DIR/store-$STAMP.tar.gz"

cd "$PROJECT_DIR"

/usr/bin/mkdir -p "$BACKUP_DIR"

/usr/bin/mysqldump \
    --defaults-extra-file=/etc/store/backup.cnf \
    store \
    > "$SQL_FILE"

/usr/bin/tar -czf "$ARCHIVE" -C "$BACKUP_DIR" "$(basename "$SQL_FILE")"

/usr/local/bin/aws \
    s3 cp \
    "$ARCHIVE" \
    s3://store-backups/

rm -f "$SQL_FILE"
```

The cron entry becomes:

```cron
0 2 * * * /opt/store/jobs/backup.sh >> /var/log/store-backup.log 2>&1
```

This script is not automatically perfect, but the hidden execution dependencies have been reduced significantly.

---

## Another realistic example: a Laravel command

A common Laravel deployment runs:

```bash
php artisan schedule:run
```

manually from the project directory.

This cron entry is fragile:

```cron
* * * * * php artisan schedule:run
```

Potential problems include:

```text
php is not in PATH
current directory is wrong
wrong PHP version is selected
.env file cannot be found
file permissions differ
application storage paths are not writable
```

A more explicit entry:

```cron
* * * * * cd /srv/myapp && /usr/bin/php artisan schedule:run >> /var/log/myapp-scheduler.log 2>&1
```

is much easier to reason about.

A wrapper script can be even cleaner:

```bash
#!/usr/bin/env bash
set -euo pipefail

cd /srv/myapp
exec /usr/bin/php artisan schedule:run
```

with:

```cron
* * * * * /srv/myapp/bin/run-scheduler
```

If PHP is provided by a custom installation:

```bash
/usr/local/php83/bin/php
```

use the intended interpreter path rather than whatever `php` happens to mean in an administrator's shell.

---

## Another realistic example: Python virtual environment and dotenv confusion

Project:

```text
/opt/collector/
├── .env
├── .venv/
└── collector.py
```

Interactive workflow:

```bash
cd /opt/collector
source .venv/bin/activate
python collector.py
```

Cron:

```cron
*/10 * * * * python /opt/collector/collector.py
```

fails with:

```text
ModuleNotFoundError: No module named 'requests'
```

The reason is not cron scheduling.

Interactive `python` points to:

```text
/opt/collector/.venv/bin/python
```

while cron resolves another interpreter.

Fix:

```cron
*/10 * * * * cd /opt/collector && /opt/collector/.venv/bin/python collector.py >> /var/log/collector.log 2>&1
```

Now imagine the program uses a dotenv library that looks for `.env` relative to the current directory. Invoking the correct Python interpreter still fails if the current directory is wrong.

That demonstrates two independent dependencies:

```text
interpreter selection
working directory
```

Good debugging isolates each one instead of changing several things randomly.

---

## Reproducing cron manually with a nearly empty environment

One of the best troubleshooting techniques is to strip the shell environment.

Suppose the job runs as `alice`:

```bash
sudo -u alice env -i \
    HOME=/home/alice \
    USER=alice \
    LOGNAME=alice \
    SHELL=/bin/sh \
    PATH=/usr/bin:/bin \
    /bin/sh -c '/opt/jobs/task.sh'
```

This is not guaranteed to perfectly reproduce every cron implementation, but it is extremely useful.

If the script fails here and works in the user's full terminal, a hidden environment dependency is likely.

Add variables one at a time.

For example:

```bash
sudo -u alice env -i \
    HOME=/home/alice \
    USER=alice \
    LOGNAME=alice \
    SHELL=/bin/sh \
    PATH=/usr/local/bin:/usr/bin:/bin \
    APP_ENV=production \
    /bin/sh -c '/opt/jobs/task.sh'
```

This method turns a vague complaint—

```text
cron is broken
```

—into a reproducible environment problem.

That is a much easier class of failure to solve.

---

## Record the exact environment on failure

A production wrapper can capture diagnostic context only when a job fails.

Example:

```bash
#!/usr/bin/env bash
set -u

LOG=/var/log/report-job.log

run_job() {
    cd /opt/reporting || return
    /opt/reporting/.venv/bin/python report.py
}

if ! run_job >>"$LOG" 2>&1; then
    rc=$?

    {
        echo "=== FAILURE $(date -Is) ==="
        echo "uid=$(id -u)"
        echo "user=$(id -un)"
        echo "pwd=$PWD"
        echo "shell=$SHELL"
        echo "path=$PATH"
        echo "--- environment ---"
        env | sort
    } >>"$LOG" 2>&1

    exit "$rc"
fi
```

There is a bug hidden in many examples of this pattern:

```bash
if ! command; then
    rc=$?
fi
```

Inside the `then` branch, `$?` may represent the status after logical negation rather than the original command status in the way the author expects.

A clearer version is:

```bash
run_job >>"$LOG" 2>&1
rc=$?

if [ "$rc" -ne 0 ]; then
    ...
    exit "$rc"
fi
```

When writing automation, clarity is more valuable than clever shell shortcuts.

---

## Use `set -x` selectively when debugging

Shell tracing can show expanded commands:

```bash
set -x
```

For example:

```bash
#!/usr/bin/env bash
set -x

echo "PATH=$PATH"
cd /opt/reporting
/opt/reporting/.venv/bin/python report.py
```

Output may reveal:

```text
+ echo PATH=/usr/bin:/bin
+ cd /opt/reporting
+ /opt/reporting/.venv/bin/python report.py
```

This is useful for debugging.

It can also leak secrets.

If a command contains:

```bash
curl -H "Authorization: Bearer $TOKEN" ...
```

`set -x` may write the token into logs.

Use tracing carefully, especially on production systems.

A safer diagnostic approach is to log only selected variables:

```bash
printf 'PATH=%s\n' "$PATH"
printf 'HOME=%s\n' "$HOME"
printf 'PWD=%s\n' "$PWD"
```

Never dump the full environment to a broadly readable log if it may contain credentials.

---

## Check executable and directory permissions along the entire path

A script may exist and still be inaccessible to the cron user.

Suppose:

```text
/opt/company/private/jobs/report.sh
```

Run:

```bash
namei -l /opt/company/private/jobs/report.sh
```

Example output:

```text
f: /opt/company/private/jobs/report.sh
drwxr-xr-x root root /
drwxr-xr-x root root opt
drwxr-x--- root company company
drwx------ admin admin private
drwxr-xr-x admin admin jobs
-rwxr-xr-x admin admin report.sh
```

Even though `report.sh` is executable, a user who cannot traverse:

```text
/opt/company/private
```

cannot reach the file.

Directory execute permission means traversal.

This is a more accurate diagnostic than looking only at:

```bash
ls -l report.sh
```

Test as the actual cron account:

```bash
sudo -u reportuser /opt/company/private/jobs/report.sh
```

If that fails, cron is not the primary issue.

---

## Environment files and security boundaries

Many applications load configuration from:

```text
.env
/etc/default/app
/etc/app/app.env
```

The file's permissions are part of the security model.

Suppose root cron executes:

```bash
. /opt/app/.env
/opt/app/backup.sh
```

and `/opt/app/.env` is writable by an unprivileged deployment user.

If `.env` is sourced by a shell, that user may insert:

```bash
malicious_command
```

and obtain execution in root's scheduled context.

Even a file intended to contain only:

```text
KEY=value
```

becomes executable shell code when sourced.

Safer options include:

```text
application-native config parser
strict env-file parser
root-owned configuration
systemd EnvironmentFile with appropriate ownership
secret-management system
```

The exact mechanism matters less than recognizing the trust boundary.

Any file that influences a privileged scheduled command must be protected against unauthorized modification.

---

## Do not let writable wrapper scripts become privilege boundaries

This cron entry is dangerous:

```cron
* * * * * root /opt/scripts/backup.sh
```

if:

```bash
ls -l /opt/scripts/backup.sh
```

shows:

```text
-rwxrwxr-x root developers ...
```

because a developer can modify code that root executes every minute.

The same problem exists if the file itself is root-owned but a parent directory is writable:

```bash
namei -l /opt/scripts/backup.sh
```

or if the script imports code from writable locations:

```bash
source /opt/app/lib/common.sh
python /opt/app/backup.py
```

A secure cron review follows the dependency chain:

```text
crontab
  -> wrapper
      -> sourced files
      -> interpreters
      -> executables
      -> configuration
      -> writable directories
```

Absolute paths reduce ambiguity, but they do not solve unsafe file ownership.

---

## `LD_PRELOAD`, language search paths, and other dangerous inherited variables

Environment variables can change program loading behavior.

Examples include:

```text
LD_PRELOAD
LD_LIBRARY_PATH
PYTHONPATH
PERL5LIB
RUBYLIB
NODE_PATH
```

A privileged automation task should not inherit untrusted values for variables that influence code loading.

For instance:

```bash
PYTHONPATH=/home/alice/devlib
```

may cause:

```bash
/usr/bin/python3 /opt/admin/report.py
```

to import attacker-controlled or development code.

Likewise, dynamic linker variables can redirect library loading for non-secure-execution contexts.

This is another reason privileged jobs should start from a minimal, controlled environment rather than from an administrator's interactive shell state.

A wrapper can explicitly define what it needs:

```bash
#!/bin/sh

PATH=/usr/bin:/bin
HOME=/root
LANG=C.UTF-8

export PATH HOME LANG

unset \
    LD_PRELOAD \
    LD_LIBRARY_PATH \
    PYTHONPATH \
    PERL5LIB \
    RUBYLIB \
    NODE_PATH

exec /usr/bin/python3 /opt/admin/report.py
```

Whether each variable should be unset depends on the application, but the principle is strong: code-loading behavior should not be accidental.

---

## A cron job should be reproducible from a plain shell

A high-quality scheduled job has a useful property:

> An administrator can reproduce its execution without reconstructing a personal login session.

For example:

```bash
sudo -u reporting env -i \
    HOME=/var/lib/reporting \
    PATH=/usr/bin:/bin \
    LANG=C.UTF-8 \
    APP_ENV=production \
    /opt/reporting/bin/daily-report
```

If that command accurately represents the job, troubleshooting becomes straightforward.

Compare that with a job requiring:

```text
login as Alice
start GNOME
open a terminal
load .bashrc
initialize nvm
unlock the SSH agent
activate a Python environment
cd into a project
export several variables manually
run the command
```

The second system has many invisible dependencies.

Cron does not create that complexity. Cron reveals it.

---

## Wrapper scripts are often the cleanest boundary

A wrapper script gives the scheduled task a defined contract.

Example:

```bash
#!/usr/bin/env bash
set -euo pipefail

PATH=/usr/local/bin:/usr/bin:/bin
HOME=/var/lib/reporting
LANG=C.UTF-8
APP_ENV=production

export PATH HOME LANG APP_ENV

umask 0027

cd /srv/reporting

exec /srv/reporting/.venv/bin/python \
    /srv/reporting/jobs/daily_report.py
```

Cron becomes:

```cron
30 1 * * * /srv/reporting/bin/run-daily-report >> /var/log/reporting/daily.log 2>&1
```

The wrapper describes:

```text
shell
failure policy
PATH
HOME
locale
application environment
umask
working directory
interpreter
entry point
```

That is much easier to review than encoding everything in one crontab line.

It also allows a direct test:

```bash
sudo -u reporting /srv/reporting/bin/run-daily-report
```

and syntax checks:

```bash
bash -n /srv/reporting/bin/run-daily-report
```

Static analysis tools such as ShellCheck can further improve shell-script quality.

---

## Keep the crontab boring

A good production crontab is often surprisingly simple.

Prefer:

```cron
*/5 * * * * /opt/app/bin/health-check
0 2 * * * /opt/app/bin/backup
15 3 * * 0 /opt/app/bin/weekly-maintenance
```

over:

```cron
*/5 * * * * cd /opt/app && export A=x && export B=y && if [ -f foo ]; then source ~/.bashrc; python something.py | awk '{print $2}' | xargs ...; fi >> /tmp/x 2>&1
```

The second version is difficult to:

```text
test
review
lint
version
debug
secure
reuse
```

Cron should describe scheduling. Scripts should describe program logic.

This separation is not merely aesthetic. It reduces the number of parsers and execution assumptions involved.

---

## A methodical troubleshooting workflow

When a cron job works manually but not automatically, use a consistent investigation sequence.

Start with the daemon:

```bash
systemctl status cron
```

or on systems using another service name:

```bash
systemctl status crond
```

Confirm the entry exists:

```bash
crontab -l
```

For another user, with sufficient privilege:

```bash
crontab -u alice -l
```

Confirm cron attempts execution through logs:

```bash
journalctl -u cron
```

or:

```bash
journalctl -u crond
```

depending on the system.

Then test identity:

```bash
sudo -u alice id
```

Capture cron's environment:

```cron
* * * * * /usr/bin/env | /usr/bin/sort > /tmp/cron-env.txt
```

Capture the working directory:

```cron
* * * * * /bin/pwd > /tmp/cron-pwd.txt
```

Capture shell behavior:

```cron
* * * * * /bin/sh -c 'echo "$0"; echo "$SHELL"' > /tmp/cron-shell.txt 2>&1
```

Test the command in a reduced environment:

```bash
sudo -u alice env -i \
    HOME=/home/alice \
    USER=alice \
    LOGNAME=alice \
    PATH=/usr/bin:/bin \
    SHELL=/bin/sh \
    /bin/sh -c '/opt/jobs/task.sh'
```

Check path resolution:

```bash
sudo -u alice env -i \
    HOME=/home/alice \
    PATH=/usr/bin:/bin \
    /bin/sh -c 'command -v python3; command -v curl; command -v aws'
```

Check filesystem traversal:

```bash
namei -l /opt/jobs/task.sh
```

Check the interpreter:

```bash
head -n 1 /opt/jobs/task.sh
```

Check syntax:

```bash
sh -n /opt/jobs/task.sh
```

or:

```bash
bash -n /opt/jobs/task.sh
```

Check output and exit code:

```bash
sudo -u alice /opt/jobs/task.sh
echo $?
```

This workflow eliminates guesswork.

---

## Lab: prove that `.bashrc` is not your cron environment

Create a test user or use a non-production account.

Add to its `.bashrc`:

```bash
export CRON_LAB_VALUE="loaded-from-bashrc"
```

Confirm interactively:

```bash
echo "$CRON_LAB_VALUE"
```

Expected:

```text
loaded-from-bashrc
```

Add:

```cron
* * * * * /usr/bin/env > /tmp/cron-lab-env.txt
```

After the job runs:

```bash
grep CRON_LAB_VALUE /tmp/cron-lab-env.txt
```

In a normal cron setup, it will probably not be present.

Now explicitly define it in crontab:

```cron
CRON_LAB_VALUE=defined-in-crontab

* * * * * /usr/bin/env > /tmp/cron-lab-env.txt
```

Check again:

```bash
grep CRON_LAB_VALUE /tmp/cron-lab-env.txt
```

Expected:

```text
CRON_LAB_VALUE=defined-in-crontab
```

The important observation is not the variable itself. It is the separation between interactive shell initialization and scheduled execution.

---

## Lab: demonstrate `/bin/sh` versus Bash

Create:

```bash
cat > /tmp/cron-shell-lab.sh <<'EOF'
#!/bin/sh

if [[ -f /etc/passwd ]]; then
    echo yes
fi
EOF

chmod +x /tmp/cron-shell-lab.sh
```

Run:

```bash
/tmp/cron-shell-lab.sh
```

On a system where `/bin/sh` does not implement `[[ ... ]]`, expect an error.

Now rewrite:

```bash
cat > /tmp/cron-shell-lab.sh <<'EOF'
#!/usr/bin/env bash

if [[ -f /etc/passwd ]]; then
    echo yes
fi
EOF
```

Run:

```bash
/tmp/cron-shell-lab.sh
```

Expected:

```text
yes
```

The shebang defines the script interpreter.

Now test command-line parsing separately:

```cron
* * * * * [[ -f /etc/passwd ]] && echo yes >> /tmp/cron-shell-command.log
```

This depends on the shell cron uses for the crontab command.

Compare with:

```cron
SHELL=/bin/bash

* * * * * [[ -f /etc/passwd ]] && echo yes >> /tmp/cron-shell-command.log
```

This lab demonstrates the difference between:

```text
shell parsing the crontab command
interpreter executing a script
```

They are related but not identical.

---

## Lab: reproduce a PATH failure

Create a private executable:

```bash
mkdir -p "$HOME/bin"

cat > "$HOME/bin/cron-hello" <<'EOF'
#!/bin/sh
echo "hello from custom path"
EOF

chmod +x "$HOME/bin/cron-hello"
```

Add the directory interactively:

```bash
export PATH="$HOME/bin:$PATH"
```

Verify:

```bash
command -v cron-hello
cron-hello
```

Now create:

```cron
* * * * * cron-hello > /tmp/cron-path-lab.txt 2>&1
```

If cron's path does not include `$HOME/bin`, the file should contain an error.

Fix it using an absolute path:

```cron
* * * * * /home/alice/bin/cron-hello > /tmp/cron-path-lab.txt 2>&1
```

Replace `alice` with the actual account name.

This is the simplest reproducible demonstration of why "it works in my terminal" is not a sufficient cron test.

---

## Lab: prove the working-directory dependency

Create:

```bash
mkdir -p /tmp/cron-cwd-lab

cat > /tmp/cron-cwd-lab/config.txt <<'EOF'
production
EOF

cat > /tmp/cron-cwd-lab/read-config.sh <<'EOF'
#!/bin/sh
cat config.txt
EOF

chmod +x /tmp/cron-cwd-lab/read-config.sh
```

Manual test:

```bash
cd /tmp/cron-cwd-lab
./read-config.sh
```

Expected:

```text
production
```

Now schedule:

```cron
* * * * * /tmp/cron-cwd-lab/read-config.sh > /tmp/cron-cwd-result.txt 2>&1
```

The script may fail:

```text
cat: config.txt: No such file or directory
```

Fix the script:

```bash
#!/bin/sh
cd /tmp/cron-cwd-lab || exit 1
cat config.txt
```

or use an absolute path:

```bash
#!/bin/sh
cat /tmp/cron-cwd-lab/config.txt
```

This lab demonstrates that an executable path and a working directory are independent properties.

---

## Lab: inspect file descriptors and TTY state

Create:

```bash
cat > /tmp/cron-fd-lab.sh <<'EOF'
#!/bin/sh

echo "PID=$$"
echo "TTY:"
tty 2>&1 || true

echo "FDs:"
ls -l /proc/$$/fd
EOF

chmod +x /tmp/cron-fd-lab.sh
```

Run interactively:

```bash
/tmp/cron-fd-lab.sh
```

Then schedule:

```cron
* * * * * /tmp/cron-fd-lab.sh > /tmp/cron-fd-result.txt 2>&1
```

Compare the outputs.

The exact file descriptors depend on the system, but the cron job should not behave like a normal terminal session.

This becomes important when troubleshooting interactive tools, buffering, and commands that inspect terminal capabilities.

---

## Lab: detect a runtime mismatch

Create a Python diagnostic:

```bash
cat > /tmp/runtime.py <<'EOF'
import os
import sys

print("executable:", sys.executable)
print("version:", sys.version.replace("\n", " "))
print("cwd:", os.getcwd())
print("HOME:", os.environ.get("HOME"))
print("PATH:", os.environ.get("PATH"))
EOF
```

Run:

```bash
python3 /tmp/runtime.py
```

Now schedule with an explicit interpreter:

```cron
* * * * * /usr/bin/python3 /tmp/runtime.py > /tmp/runtime-cron.txt 2>&1
```

Compare:

```bash
cat /tmp/runtime-cron.txt
```

If the interactive shell uses pyenv, Conda, or a virtual environment, the interpreter paths may differ.

This lab is especially useful because runtime mismatches can produce subtle dependency differences without an obvious "command not found" error.

---

## Design rules that survive real production systems

A scheduled command is easier to operate when it follows a few strict rules.

Use explicit executable paths for critical commands.

Use a controlled `PATH` rather than an interactive user's long path.

Do not assume `.bashrc`, `.profile`, or runtime-manager initialization has occurred.

Do not assume the working directory.

Do not depend on an interactive terminal.

Do not depend on an ephemeral SSH or desktop session.

Choose the required shell intentionally.

Keep complicated shell logic in version-controlled scripts.

Use a deterministic interpreter for Python, Node.js, Ruby, PHP, or other language runtimes.

Make locale and timezone assumptions explicit when they influence data.

Protect configuration and environment files according to the privilege level of the scheduled job.

Do not expose secrets through command lines, debug tracing, logs, or world-readable files.

Preserve meaningful exit statuses.

Make failures observable.

Test using the exact user account.

Test with a reduced environment.

Inspect rather than assume.

---

## Final perspective

Cron itself is relatively small in concept. It watches time, evaluates schedule definitions, and starts commands.

Most difficult cron incidents happen after scheduling has already succeeded.

The process starts, but:

```text
the wrong shell parses the command
the executable cannot be found
the runtime version is different
the current directory is wrong
the configuration file is missing
the locale changes program behavior
credentials exist only in an interactive session
the command expects a TTY
stdout and stderr disappear
a pipeline masks the real error
a writable dependency creates a security vulnerability
```

These are execution-environment problems.

The strongest way to design cron jobs is to minimize inherited state. A job should know which user it runs as, which shell or interpreter it uses, where it starts, which environment values it requires, which files it trusts, where output goes, and how failure is represented.

When those properties are explicit, cron becomes boring.

That is a good thing.

A boring cron job is easy to reproduce, easy to audit, easy to secure, and easy to repair at three in the morning.
