# MariaDB Service Failure — Root Cause and Fix

**Environment:** Production database server, RHEL/CentOS-based
**Service affected:** `mariadb.service`
**Impact:** Application unable to connect to the database

## Symptom

The `mariadb` service was inactive on the database server. The application couldn't reach the database, and manual connection attempts from the CLI failed with:

```
Can't connect to local MySQL server through socket '/var/lib/mysql/mysql.sock' (2)
```

## Diagnosis Path

### 1. Check service status

```bash
sudo systemctl status mariadb
```

Result: `inactive (dead)`, not a live crash. This meant the service never got past initialization, rather than crashing mid-run, an important distinction, it points the investigation toward startup rather than runtime behavior.

### 2. Attempt to start the service

```bash
sudo systemctl start mariadb
```

Result: failed with a generic control-process error, pointing to `journalctl` and `systemctl status` for detail. This top-level message rarely reveals the real problem on its own.

### 3. Inspect the journal for the unit

```bash
sudo journalctl -xeu mariadb
```

This is where the real answer showed up:

```
mariadb-prepare-db-dir: Cannot change ownership of the
database directories to the user.
Initialization of MariaDB database failed.
```

Systemd's own journal frequently contains the true root cause before the application ever gets a chance to log anything itself, worth checking before chasing application-specific log files.

### 4. Confirm root cause via directory ownership

```bash
sudo ls -l /var/lib/
```

Finding: `/var/lib/mysql` was owned by `root:mysql` instead of `mysql:mysql`. MariaDB's startup process attempts to take ownership of its data directory before initializing, and fails if it can't.

## Root Cause

Incorrect ownership on the MariaDB data directory (`/var/lib/mysql`) prevented the `mariadb-prepare-db-dir` startup step from completing, which in turn prevented the `mariadb` service from starting at all.

## Fix Applied

```bash
sudo chown -R mysql:mysql /var/lib/mysql
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

Service came up successfully:

```
Active: active (running)
```

## Key Lesson

`chown` without `-R` only touches the named object, not its contents.

The first attempt (`chown mysql:mysql /var/lib/mysql`, no `-R`) corrected only the top-level directory. Everything inside remained misowned, and the service still failed to start. Adding the recursive flag was what actually resolved it, a reminder to check whether a fix actually reached everything it needed to, not just the object you named.

## Post-Fix Observations (Not Errors)

- Connecting as a normal Linux user returned `Access denied`, expected. MariaDB's default root access uses `unix_socket` authentication, tied to the Linux `mysql`/`root` system accounts, not arbitrary Linux users.
- Running the client via `sudo` succeeded, because `sudo` executes as `root`, and socket authentication maps the Linux `root` user to MariaDB's built-in root account.

## Verifying the Fix Is Complete

A running service isn't sufficient confirmation on its own. To confirm the incident is genuinely resolved:

```sql
SHOW DATABASES;
SELECT User, Host FROM mysql.user;
```

Then have the application (or support team) retest the actual application-to-database connection, that's the real definition of "resolved," not just the daemon reporting active.

## General Troubleshooting Habit

When a systemd-managed service fails to start, check `journalctl -xeu <service>` before chasing application log files. The `ExecStartPre` output often contains the root cause before the application has logged anything at all.
