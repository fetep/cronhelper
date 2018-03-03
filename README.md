# cronhelper

## Features

* Directly send cronmail instead of falling back to cron's built-in mail.
This allows for better email formatting and customization.

* Only send cronmail if the process exits non-zero

* Optionally records status information to a local status file. Nagios
compatible script `check_cronhelper` is included.

* Jitter mode to avoid too many jobs starting at once.

* Locking to prevent jobs from piling up.

* Monitoring status file to track job performance and exit codes.

## Usage

```
  cronhelper [options] cronid command

  Options:
    -b BODY            cronmail body format
    -f FROM            cronmail from address format
    -h                 show this help message
    -i                 ignore exit code when deciding to send email
    -j JITTER          sleep 0-JITTER seconds before executing
    -k COUNT           keep COUNT job history records in the status file
    -m ADDR,ADDR,...   cronmail recipients (default: $USER)
    -S SHELL           run command under SHELL
    -s SUBJ            cronmail subject format
    -W                 disable writing to the status file
    -x                 hold an exclusive lock for cronid
```

### General Options

* `cronid`: required to write monitoring status information and use exclusive
  locking. Most of the time, this should be unique per job. For some locking
  situations, different jobs may want the same `cronid` with `-x` to avoid
  running at the same time.

* `command`: executed as a shell command

* `-j JITTER`: enable jitter mode. sleep between 0 and `JITTER` seconds before
  running `command`. Specify the `m` suffix for minutes (e.g. `-j 15m`).

* `-S SHELL`: run `command` under `SHELL` (defaults to `bash`)

* `-x`: exclusive lock - only allow one job as `cronid` to run at a time

### Cron Mail Options

See [[#Cron_Mail]]

* `-b BODY`: the cronmail body. Available printf-style sequences:

| Sequence | Description |
| :--- | :--- |
| %? | `command` exit code |
| %c | `command` |
| %e | `command` stderr |
| %O | `command` output (both stdout + stderr) |
| %o | `command` stdout |

The default body is `%c exited %?:\n\n%O`.

* `-f FROM`: the cronmail from address. Available printf-style sequences:

| Sequence | Description |
| :--- | :--- |
| %h | hostname |
| %u | username |

The default from address is `Cron Helper <%u@%h>`.

* `-i`: ignore exit code when deciding to send email. By default, if `command`
  exits `0`, cronmail will not be sent.

* `-k COUNT`: keep `COUNT` job history records in the monitoring status file,
  defaults to `10`. This may need to be increased if you want to monitor percent
  job success over a larger sample size (see [[#Monitoring]]).

* `-m ADDR,ADDR,...`: comma separated list of cronmail recipients

* `-s SUBJECT`: the cronmail subject. Available printf-style sequences:

| Sequence | Description |
| :--- | :--- |
| %? | `command` exit code |
| %c | `command` |
| %h | hostname |
| %i | `cronid` |
| %u | username |

The default subject is `Cron <%u@%h> [%i] %c`

## Cron Mail

cronhelper never prints to stdout or stderr, so the system cron daemon will
not send email.

The mail format is customizable, see [[#Cron_Mail_Options]].

Extra headers:

* `X-Cron-Id: CRONID`: the `cronid` specified on the command line
* `X-Cron-Env: <ENV=value>`: one header per env var
* `X-Cron-Exit: EXITCODE`: command exit code

## Monitoring

A collection of cronhelper status files (one per `cronid`) are stored in
`~/.cronhelper/status`. Note that this requires every user that runs
`cronhelper` to have a writable home directory. Writes to the monitoring
status file can be disabled with `-W`.

Job history records for the last 10 runs (override with `-k COUNT`) are
stored. Each record includes:

| Key | Description |
| :--- | :--- |
| ? | `command` exit code |
| c | command |
| e | end time (seconds since epoch) |
| s | start time (seconds since epoch) |

TODO: deal with multiple jobs w/same cronid writing status data at once.
How long to wait for a lock, and what happens when it fails.

### Nagios Integration

```
check_cronhelper [-e] [-w TIME]

  -e            alert if the exit code is non-zero
  -w WARN,CRIT  alert if process has not run within WARN,CRIT seconds
```
