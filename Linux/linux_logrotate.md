# logrotate

> A complete training guide to Linux's log file rotation utility — rotation, compression, removal, and mailing of system logs.

`logrotate · Linux System Administration`

---

## Contents

- [01 What Is Logrotate?](#01-what-is-logrotate)
- [02 Why Use Logrotate?](#02-why-use-logrotate)
- [03 How It Works](#03-how-it-works)
- [04 Command-Line Usage](#04-command-line-usage)
- [05 Configuration File Structure](#05-configuration-file-structure)
- [06 Configuration Directives Reference](#06-configuration-directives-reference)
- [07 Scripts & Hooks](#07-scripts--hooks)
- [08 Linux Services & Integration](#08-linux-services--integration)
- [09 Practical Examples](#09-practical-examples)
- [10 Best Practices](#10-best-practices)

---

## 01 What Is Logrotate?

`logrotate` is a standard Linux utility designed to simplify the administration of systems that generate large volumes of log files. It automates the lifecycle management of logs: rotating them on a schedule (or by size), compressing old logs to save disk space, removing aged-out files, and optionally mailing them to administrators.

**Core Capabilities**

- **Rotation** — Rename current log and start a fresh one, keeping a configurable number of old copies.
- **Compression** — Compress rotated logs with gzip (default), bzip2, xz, or any custom compressor.
- **Removal** — Automatically delete logs older than a defined age or count.
- **Mailing** — Email log files to a specified address before they are removed.
- **Script execution** — Run pre-rotation and post-rotation shell scripts (e.g., to restart a service).

---

## 02 Why Use Logrotate?

| Benefit | Description |
|---|---|
| **Prevents Disk Exhaustion** | Unmanaged logs can fill a filesystem and cause service outages. Logrotate enforces size and age limits. |
| **Preserves History** | Keeps a controlled number of rotated/compressed copies for forensics and auditing purposes. |
| **Automates Maintenance** | Runs unattended via cron/systemd timer — no manual intervention needed. |
| **Per-Service Configuration** | Each application can have its own rotation rules (frequency, retention, scripts). |

---

## 03 How It Works

Logrotate is not a daemon — it is a command-line tool that is invoked periodically (typically once per day) by `cron` or a `systemd timer`.

```
[Step 1] Cron / Timer Fires  →  [Step 2] Read Config Files  →  [Step 3] Check Criteria  →  [Step 4] Rotate & Compress  →  [Step 5] Run Scripts
```

### State File

Logrotate tracks when each log was last rotated in a **state file** (default: `/var/lib/logrotate/logrotate.status`). This allows time-based rotation decisions (daily, weekly, monthly) to survive reboots and skipped cron runs. The state file is locked during execution to prevent parallel runs from conflicting.

---

## 04 Command-Line Usage

```
logrotate [OPTIONS] config_file [config_file2 ...]
```

| Option | Description |
|---|---|
| `-f, --force` | Force rotation even if logrotate doesn't think it is necessary. |
| `-d, --debug` | Dry-run mode. No changes are made; only debug messages are printed. |
| `-v, --verbose` | Display informational messages during rotation. |
| `-s, --state FILE` | Use an alternate state file instead of the default. |
| `--skip-state-lock` | Do not lock the state file (for environments where locking is unsupported). |
| `-l, --log FILE` | Write verbose output to the specified log file (overwritten each run). |
| `-m, --mail CMD` | Specify the mail command (default: `/bin/mail`). |
| `--version` | Print version information. |

### Common Invocations

```bash
# Normal daily run (how cron calls it)
/usr/sbin/logrotate /etc/logrotate.conf

# Dry-run to test a new configuration
logrotate -d /etc/logrotate.d/myapp

# Force rotation with verbose output
logrotate -fv /etc/logrotate.d/nginx

# Use a custom state file (useful for non-root users)
logrotate -s ~/.logrotate.status ~/myapp-logrotate.conf
```

---

## 05 Configuration File Structure

Logrotate configuration consists of **global directives** (applied to all log blocks unless overridden) and **log-specific blocks** enclosed in curly braces.

### File Locations

| Path | Purpose |
|---|---|
| `/etc/logrotate.conf` | Main configuration file. Sets global defaults and includes the drop-in directory. |
| `/etc/logrotate.d/` | Drop-in directory. Each installed package places its own config file here. |
| `/var/lib/logrotate/logrotate.status` | State file tracking last rotation timestamps. |

### Anatomy of a Configuration File

```
# Global options (apply to all blocks below unless overridden)
weekly
rotate 4
create
dateext
compress

# Include per-application configs
include /etc/logrotate.d

# A specific log block
/var/log/wtmp {
    monthly
    create 0664 root utmp
    minsize 1M
    rotate 1
}
```

> **Key rules:** Comments start with `#`. Later config files override earlier ones. If a directory is given on the command line, every file inside it is treated as a config file. Values are separated from directives by whitespace and/or `=`.

---

## 06 Configuration Directives Reference

### Rotation Control

| Directive | Description |
|---|---|
| `rotate count` | Keep *count* rotated copies. Default: 0. `0` = delete immediately. `-1` = never delete. |
| `olddir directory` | Move rotated logs to this directory (must be on same device unless using copy/copytruncate). |
| `noolddir` | Rotate logs in-place (overrides olddir). |
| `su user group` | Rotate files as this user/group instead of root. |

### Frequency

| Directive | Description |
|---|---|
| `hourly` | Rotate every hour (requires hourly cron). |
| `daily` | Rotate every day. |
| `weekly [weekday]` | Rotate weekly. 0=Sun ... 6=Sat, 7=every 7 days. |
| `monthly` | Rotate on the first run of each month. |
| `yearly` | Rotate if the year has changed. |
| `size N[k\|M\|G]` | Rotate when file exceeds this size (ignores time schedule). |

### File Selection

| Directive | Description |
|---|---|
| `missingok` | Don't error if the log file is missing. |
| `nomissingok` | Error if the log file is missing (default). |
| `notifempty` | Skip rotation if the log is empty. |
| `ifempty` | Rotate even if the log is empty (default). |
| `minage days` | Don't rotate logs younger than this. |
| `maxage days` | Delete rotated logs older than this. |
| `minsize N` | Rotate only if size AND time criteria are met. |
| `maxsize N` | Rotate if size is exceeded, even before time interval. |

### File Creation & Handling

| Directive | Description |
|---|---|
| `create mode owner group` | Create a new empty log file after rotation with specified permissions. |
| `nocreate` | Don't create a new log file after rotation. |
| `copy` | Copy the log but don't modify the original. Useful for snapshots. |
| `copytruncate` | Copy the log, then truncate the original to zero. For apps that can't reopen log files. |
| `renamecopy` | Rename log to .tmp, run postrotate, then copy to final name. |
| `shred / noshred` | Securely delete old logs using `shred -u` instead of unlink. |

### Compression

| Directive | Description |
|---|---|
| `compress / nocompress` | Enable/disable gzip compression on rotated files. |
| `compresscmd cmd` | Use a custom compressor (e.g., bzip2, xz). |
| `compressext ext` | Custom extension for compressed files. |
| `compressoptions opts` | Options passed to compressor. Default: `-6` for gzip. |
| `delaycompress` | Don't compress until the next rotation cycle. Useful if apps keep writing to the old file briefly. |
| `uncompresscmd cmd` | Command to decompress (default: gunzip). |

### Filename Customization

| Directive | Description |
|---|---|
| `dateext / nodateext` | Use date-based extensions (YYYYMMDD) instead of numeric. |
| `dateformat fmt` | strftime-style format for date extensions. Must be lexically sortable. |
| `dateyesterday` | Use yesterday's date for the extension name. |
| `start N` | Start numbering rotated files at N (default: 1). |
| `extension ext` | Preserve a file extension after rotation (e.g., `.log`). |
| `addextension ext` | Append an extension to rotated files. |

### Mail

| Directive | Description |
|---|---|
| `mail address` | Mail log to this address when it is about to be removed. |
| `nomail` | Don't mail old logs. |
| `mailfirst` | Mail the just-rotated file. |
| `maillast` | Mail the about-to-expire file (default). |

---

## 07 Scripts & Hooks

Logrotate can execute shell scripts at various points during the rotation lifecycle. Scripts run between a starting keyword and `endscript`, executed via `/bin/sh`.

| Directive | When It Runs | Argument |
|---|---|---|
| `firstaction` | Once before any matching log is rotated. | Whole glob pattern. |
| `prerotate` | Before each log file is rotated. | Absolute path of the log (or pattern if `sharedscripts`). |
| `postrotate` | After each log file is rotated. | Absolute path of original + rotated file (or pattern if `sharedscripts`). |
| `lastaction` | Once after all matching logs are rotated. | Whole glob pattern. |
| `preremove` | Just before a log file is removed. | Name of the file about to be deleted. |

### sharedscripts vs nosharedscripts

> **sharedscripts** — The `prerotate` and `postrotate` scripts run only **once** per block, even if the block matches multiple files (e.g., a wildcard pattern). The whole pattern is passed as the first argument. If an error occurs in any script, remaining actions are skipped for *all* matching logs.

> **nosharedscripts (default)** — Scripts run once **per log file**. Each invocation receives the absolute path of the specific file being rotated. An error in a script only stops further actions for that one file.

---

## 08 Linux Services & Integration

Logrotate itself is not a service. It relies on and integrates with the following system components:

| Service / Component | Role |
|---|---|
| `cron` (crond) | The traditional scheduler. A daily cron job at `/etc/cron.daily/logrotate` invokes logrotate once per day. This is the most common trigger mechanism. |
| `systemd timer` | On modern distributions (RHEL/CentOS 7+, Fedora, Ubuntu 16.04+), a systemd timer (`logrotate.timer` / `logrotate.service`) replaces the cron job. Check with `systemctl list-timers`. |
| `/bin/sh` | All embedded scripts (prerotate, postrotate, etc.) are executed by the system shell. |
| `/bin/mail` | Default mail transport used to email rotated logs. Can be overridden with the `-m` flag or the `mail` directive. |
| `gzip` / `gunzip` | Default compression/decompression utilities. Configurable via `compresscmd` and `uncompresscmd`. |
| `shred` | GNU coreutils secure deletion tool, optionally used when the `shred` directive is enabled. |
| `SELinux / ACLs` | On systems with SELinux, logrotate must operate within its security context. The `create` directive sets ownership/mode; the `su` directive controls execution identity. |

### Checking Your Trigger Mechanism

```bash
# Check if a systemd timer is active
systemctl status logrotate.timer

# Check if cron is being used instead
ls -lha /etc/cron.daily/logrotate

# See timer schedule
systemctl list-timers --all | grep logrotate
```

---

## 09 Practical Examples

### Example 1: Basic Web Server Logs

```
# /etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        # Signal nginx to reopen log files
        [ -f /var/run/nginx.pid ] && kill -USR1 $(cat /var/run/nginx.pid)
    endscript
}
```

**What this does** — Rotates all `.log` files in `/var/log/nginx/` daily. Keeps 14 compressed copies. Delays compression by one cycle (so the most recent rotated file stays uncompressed for tools that may still be reading it). After rotation, sends `USR1` to nginx so it reopens its log file handles. The script runs only once for all matched files (`sharedscripts`).

---

### Example 2: Application with copytruncate

```
# /etc/logrotate.d/myapp
/opt/myapp/logs/app.log {
    size 50M
    rotate 10
    compress
    copytruncate
    missingok
    notifempty
    su appuser appgroup
    olddir /opt/myapp/logs/archive
    createolddir 0750 appuser appgroup
}
```

**What this does** — For applications that cannot reopen their log file (they hold a persistent file descriptor), `copytruncate` copies the log and then truncates the original to zero bytes. Rotation triggers when the file exceeds 50 MB. Old logs are moved to an `archive` subdirectory (created automatically if it doesn't exist). The `su` directive ensures operations run as `appuser:appgroup`.

---

### Example 3: Date-Based Rotation with Mailing

```
# /etc/logrotate.d/audit
/var/log/audit/audit.log {
    monthly
    rotate 12
    compress
    dateext
    dateformat -%Y%m%d
    maxage 365
    mail admin@example.com
    maillast
    create 0600 root root
    postrotate
        /sbin/service auditd restart
    endscript
}
```

**What this does** — Rotates the audit log monthly, keeping 12 copies. Files are named with date extensions (e.g., `audit.log-20260301.gz`). Logs older than 365 days are mailed to the administrator before deletion. The audit daemon is restarted after rotation.

---

### Example 4: High-Volume Microservice

```
# /etc/logrotate.d/microservice
/var/log/microservice/*.log {
    hourly
    rotate 168            # 7 days * 24 hours
    missingok
    notifempty
    compress
    compresscmd /usr/bin/xz
    compressext .xz
    uncompresscmd /usr/bin/unxz
    dateext
    dateformat -%Y%m%d%H
    maxsize 200M
    sharedscripts
    postrotate
        /usr/bin/systemctl reload microservice
    endscript
}
```

**What this does** — Handles high-volume logs that need hourly rotation. Uses xz compression for better ratios. Also triggers if any single file exceeds 200 MB before the hour is up (`maxsize`). Keeps a week of hourly backups (168 copies). Requires the cron/timer to run hourly, not just daily.

---

### Example 5: Complete /etc/logrotate.conf

```
# /etc/logrotate.conf - Global configuration

# Rotate weekly by default
weekly

# Keep 4 weeks of old logs
rotate 4

# Create new (empty) log files after rotating old ones
create

# Use date-based extensions
dateext

# Compress rotated logs
compress

# Include per-package configs
include /etc/logrotate.d

# System logs that aren't covered by packages
/var/log/wtmp {
    monthly
    create 0664 root utmp
    minsize 1M
    rotate 1
}

/var/log/btmp {
    missingok
    monthly
    create 0600 root utmp
    rotate 1
}
```

---

## 10 Best Practices

**1. Always Test First**
Run `logrotate -d /etc/logrotate.d/myapp` (debug/dry-run mode) before deploying any new configuration. This validates syntax and shows what would happen without making changes.

**2. Use missingok and notifempty**
These prevent unnecessary errors and wasted rotation cycles. A missing log shouldn't crash your rotation, and an empty log doesn't need rotating.

**3. Use sharedscripts for Wildcard Patterns**
When a block matches multiple files (e.g., `/var/log/nginx/*.log`), use `sharedscripts` so the postrotate script (e.g., service reload) runs only once instead of once per file.

**4. Prefer delaycompress with Long-Running Processes**
Some processes may still write to the previous log file briefly after rotation. `delaycompress` keeps the most recent rotated file uncompressed so these writes don't fail.

**5. Use the su Directive for Non-Root Directories**
If logrotate runs as root but the log directory is owned by an application user, use `su user group` to avoid permission issues and follow the principle of least privilege.

**6. Be Careful with Wildcards**
A bare `*` wildcard will match previously rotated files too. Use specific patterns like `*.log` and/or use `olddir` to move rotated files out of the match path.

> **Warning:** `copytruncate` has a small race condition — data written between the copy and the truncate is lost. Only use it when the application truly cannot be told to reopen its log file. Prefer `create` + a postrotate signal (e.g., `SIGHUP`) when possible.

---

*Logrotate Training Document · prepared by oramad · 20-March-2026*

Project: https://github.com/logrotate/logrotate
