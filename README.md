# ⏰ The Complete Guide to Cron

Scheduling, syntax, real-world use cases, and best practices for the Unix job scheduler.

## Contents

1. [What Is Cron?](#1-what-is-cron)
2. [Crontab Syntax](#2-crontab-syntax)
3. [Special Strings & Characters](#3-special-strings--characters)
4. [Managing Crontabs](#4-managing-crontabs)
5. [Schedule Examples](#5-schedule-examples)
6. [Real-World Use Cases](#6-real-world-use-cases)
7. [Environment & Gotchas](#7-environment--gotchas)
8. [Best Practices](#8-best-practices)
9. [Alternatives to Cron](#9-alternatives-to-cron)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. What Is Cron?

**Cron** is a time-based job scheduler built into Unix-like operating systems (Linux, macOS, BSD). It runs the background daemon `crond`, which wakes up every minute, checks a set of configuration files called **crontabs**, and executes any command whose schedule matches the current time.

Cron has existed since Version 7 Unix (1975) and remains the default way to automate recurring tasks on servers: backups, log rotation, report generation, cleanup scripts, health checks, and more. Its enduring appeal is simplicity — a one-line schedule expression plus a shell command.

---

## 2. Crontab Syntax

Each line in a crontab has five time fields followed by the command to run:

```
┌───────────── minute (0–59)
│ ┌───────────── hour (0–23)
│ │ ┌───────────── day of month (1–31)
│ │ │ ┌───────────── month (1–12)
│ │ │ │ ┌───────────── day of week (0–6, Sunday=0 or 7)
│ │ │ │ │
* * * * * command-to-execute
```

For example:

```
30 2 * * * /usr/bin/backup.sh
```

runs `backup.sh` every day at **2:30 AM**.

| Field | Allowed values | Notes |
|---|---|---|
| Minute | 0–59 | |
| Hour | 0–23 | 24-hour clock, no AM/PM |
| Day of month | 1–31 | |
| Month | 1–12 or JAN–DEC | |
| Day of week | 0–6 or SUN–SAT | 0 and 7 both mean Sunday |

---

## 3. Special Strings & Characters

- **`*`** — any value. Matches every possible value in that field.
- **`,`** — list. Specifies multiple values, e.g. `1,15` = the 1st and 15th.
- **`-`** — range. `MON-FRI` = Monday through Friday.
- **`/`** — step. `*/15` = every 15 units (minutes, hours, etc).

Nicknames some cron implementations support instead of the 5 fields:

| Nickname | Equivalent |
|---|---|
| `@yearly` / `@annually` | `0 0 1 1 *` |
| `@monthly` | `0 0 1 * *` |
| `@weekly` | `0 0 * * 0` |
| `@daily` / `@midnight` | `0 0 * * *` |
| `@hourly` | `0 * * * *` |
| `@reboot` | Run once at daemon/system startup |

---

## 4. Managing Crontabs

```bash
crontab -e          # edit your personal crontab
crontab -l          # list your current crontab
crontab -r          # remove your crontab (careful!)
crontab -u bob -e   # edit another user's crontab (needs privileges)
```

System-wide locations:

- `/etc/crontab` — system crontab, includes a *user* field before the command
- `/etc/cron.d/` — drop-in files, same 6-field format, often installed by packages
- `/etc/cron.daily`, `cron.hourly`, `cron.weekly`, `cron.monthly` — directories of scripts run by `run-parts` at the matching cadence

---

## 5. Schedule Examples

| Expression | Meaning |
|---|---|
| `0 * * * *` | Every hour, on the hour |
| `*/5 * * * *` | Every 5 minutes |
| `0 0 * * *` | Every day at midnight |
| `0 9 * * 1-5` | 9 AM, Monday through Friday |
| `0 0 1 * *` | Midnight on the 1st of every month |
| `0 0 * * 0` | Midnight every Sunday |
| `0 */4 * * *` | Every 4 hours |
| `0 22 * * 1-5` | 10 PM on weeknights |
| `30 6 1,15 * *` | 6:30 AM on the 1st and 15th |
| `0 0 1 1 *` | Midnight, January 1st (yearly) |

---

## 6. Real-World Use Cases

### Systems & infrastructure

**Backups** — Nightly database dumps or file-system snapshots to local disk or offsite storage.
```
0 2 * * * pg_dump mydb | gzip > /backups/db-$(date +\%F).sql.gz
```

**Log rotation** — Trim, compress, or archive logs before they fill the disk (often via `logrotate` triggered by cron).
```
0 0 * * * /usr/sbin/logrotate /etc/logrotate.conf
```

**Disk cleanup** — Delete temp files, old build artifacts, or stale cache entries.
```
0 3 * * * find /tmp -mtime +7 -delete
```

**SSL/TLS renewal** — Renew Let's Encrypt certificates before expiry.
```
0 3 * * * certbot renew --quiet
```

**System updates** — Pull security patches on a schedule (with caution on production).
```
0 4 * * 0 apt-get update && apt-get -y upgrade
```

**Health checks** — Ping a service or endpoint and alert if it's down.
```
*/2 * * * * /scripts/healthcheck.sh
```

### Data & application workflows

**ETL / data pipelines** — Pull data from an API or warehouse, transform it, and load it downstream on a fixed cadence.
```
0 1 * * * python etl_job.py
```

**Cache warming** — Pre-populate caches or CDNs before peak traffic hours.
```
0 5 * * * /scripts/warm_cache.sh
```

**Report generation** — Compile daily/weekly analytics and email or upload them.
```
0 7 * * 1 /scripts/weekly_report.py
```

**Queue/worker maintenance** — Requeue stuck jobs, clear dead-letter queues, restart stalled workers.
```
*/10 * * * * /scripts/requeue_failed_jobs.sh
```

**Search index rebuilds** — Refresh Elasticsearch/Solr indexes from source-of-truth data.
```
0 */6 * * * /scripts/reindex.sh
```

**Session/token cleanup** — Purge expired sessions, tokens, or password-reset links from the DB.
```
15 * * * * php artisan sessions:prune
```

### Business & communication

**Email digests** — Send daily/weekly summary emails or newsletters.
```
0 8 * * * /scripts/send_digest.py
```

**Invoice/billing runs** — Generate invoices or run subscription billing on the 1st of the month.
```
0 6 1 * * /scripts/run_billing.py
```

**Social media posting** — Auto-publish scheduled posts stored in a queue.
```
*/30 * * * * /scripts/post_scheduled.py
```

**Reminders & follow-ups** — Notify users of upcoming deadlines, trial expirations, renewals.
```
0 9 * * * /scripts/send_reminders.py
```

### Security & compliance

**Vulnerability scans** — Run nightly scans against servers or dependencies.
```
0 1 * * * /scripts/security_scan.sh
```

**Audit log shipping** — Ship logs to a SIEM or long-term archive on a schedule.
```
0 * * * * /scripts/ship_audit_logs.sh
```

**Account/permission audits** — Flag stale accounts or over-privileged roles weekly.
```
0 5 * * 1 /scripts/access_audit.py
```

> **Rule of thumb:** if a task is repetitive, time-based, and doesn't need a human to click "go," it's a cron candidate.

---

## 7. Environment & Gotchas

- **Minimal environment:** cron jobs run with a stripped-down `PATH` and no shell profile loaded — commands that work in your interactive terminal may fail silently under cron. Use absolute paths (`/usr/bin/python3`, not `python3`) or set `PATH` explicitly at the top of the crontab.
- **Working directory:** jobs start in the user's home directory, not wherever you were when you wrote the script. Use absolute paths for input/output files too.
- **No output = no mail (unless configured):** by default cron emails stdout/stderr to the local user's mailbox, which often nobody reads. Redirect to a log file: `>> /var/log/job.log 2>&1`.
- **Time zone:** cron uses the system's local time zone by default (some setups set `CRON_TZ` or `TZ` per-line). Verify with `date` on the host, especially across cloud regions.
- **Day-of-month AND day-of-week:** if both fields are restricted (not `*`), cron treats it as an OR, not AND — a common source of surprise.
- **Daylight saving time:** jobs scheduled during the "spring forward" hour may be skipped; jobs in the "fall back" hour may run twice.
- **Overlapping runs:** cron does not check whether the previous invocation is still running. A slow job can pile up duplicate instances. Guard with a lock file or `flock`.

---

## 8. Best Practices

1. **Use `flock` to prevent overlaps:** `* * * * * flock -n /tmp/job.lock /scripts/job.sh`
2. **Log everything** with timestamps, and rotate those logs too.
3. **Fail loudly:** have scripts exit non-zero on failure and alert (Slack/email/PagerDuty) rather than fail silently.
4. **Keep crontab entries thin:** call a wrapper script rather than embedding long logic inline.
5. **Version-control your cron scripts** and deploy the crontab itself as code (e.g. via Ansible, Chef, or a dotfiles repo) rather than editing production by hand.
6. **Add comments** (lines starting with `#`) explaining what each job does and why.
7. **Monitor for "job never ran"** with a dead-man's-switch service (e.g. Healthchecks.io, Cronitor) that pages you if a scheduled ping doesn't arrive.
8. **Test schedules** with tools like crontab.guru before deploying.

---

## 9. Alternatives to Cron

| Tool | Best for |
|---|---|
| systemd timers | Modern Linux servers; better logging, dependency management, and `OnCalendar=` syntax integrated with systemd units |
| Windows Task Scheduler | Native Windows scheduling with a GUI |
| Airflow / Dagster / Prefect | Complex data pipelines with dependencies, retries, and DAGs |
| Kubernetes CronJob | Scheduled jobs inside a Kubernetes cluster |
| Cloud schedulers (AWS EventBridge, GCP Cloud Scheduler) | Serverless / managed environments |
| Celery beat | Scheduling tasks within an existing task-queue app |

---

## 10. Troubleshooting

- **Job didn't run:** check `grep CRON /var/log/syslog` (Debian/Ubuntu) or `journalctl -u cron` / `journalctl -u crond` (systemd-based).
- **Runs manually but not via cron:** almost always a `PATH` or environment-variable difference — reproduce with `env -i /bin/bash -c 'your-command'`.
- **Permission errors:** confirm the crontab's owning user can access the files/binaries referenced.
- **Wrong day/time triggered:** double check the day-of-month/day-of-week OR behavior and the server's time zone.

---

*Reference guide — verify current syntax against your OS's `man 5 crontab`, since minor dialect differences exist between cron implementations (Vixie cron, cronie, systemd-cron).*
